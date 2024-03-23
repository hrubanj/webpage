---
layout: post
title: "Building an AI-powered Power Plant - Part 2"
date: 2024-03-18 00:00:00 +0000
categories: [ programming, optimization, electricity, ml, ai ]
---
<link href="//maxcdn.bootstrapcdn.com/font-awesome/4.2.0/css/font-awesome.min.css" rel="stylesheet">
<a href="{{ site.baseurl }}/index.html"><i class='fa fa-home'></i> Home</a>

### Introduction
In this post, explore how we improved our solution for controlling our home power plant.
As a brief reminder, we have solar panels and a battery. We can consume produced energy or sell it to the grid.

[Previously]({% post_url 2024-02-24-controlling-home-power-plant-with-ai-part-1 %}), we introduced a simple
scheduler that let us preset when to charge the battery and when to discharge it. We learned that this
was not quite what we needed. We have to add optimization, so it gives users something that they can't
easily do themselves.

In the following sections, we will discuss how our understanding of the problem evolved, we'll introduce our optimization
approach, and we'll discuss predictions for variables going into the optimization. Next, we'll showcase our rudimentary UI, and
discuss how it all works together.

### New learnings
In the previous post, we briefly mentioned that electricity spot prices can sometimes be negative. That is bad if
you are selling electricity, but good if you are buying it. Economically, it might even make sense to buy and waste
the energy if you get paid for it.

This turns out to be a bit more complicated. If you are selling for a spot price, you receive the spot price minus a 
feed to the grid operator. If you are buying, you pay the spot price plus several fees that usually almost double the price.
Even if the spot price is negative, the final price never is. It can be in theory, but it is so unlikely, that
it is not worth considering.

Luckily, this does not change our understanding of the problem. In fact, as we were digging deeper into it,
I was quite happy to learn that we got the fundamentals right.

### Optimization
Let us first express the problem in somewhat formal terms.

We have three binary switches:
- `supply_to_grid` - whether we are selling electricity to the grid
- `charge_from_grid` - whether we are charging the battery from the grid
- `panels_on` - whether the solar panels are activated
Our goal is to decide when these switches should be on for several future time periods.

Heuristically, we want to supply to grid when the price is high, and we have enough electricity to spare,
We want to charge from the grid almost only when there is no solar energy and our battery would otherwise get
discharged so much that it might get damaged.
We want to have the panels on almost always, except when our battery is full and electricity selling price is negative.

We can express a cost function using the above variables, buying and selling prices, consumption, and solar power production.
Then, we impose constraints on the variables, e.g. that we cannot charge battery to more than its maximum capacity.
Summing cost for all periods will give us the total cost. We want to minimize it.

Having formalized the problem, we can plug it into linear programming solver, and let it pick the best values for us.
We opted for Google OR-Tools, as it is free, and it has a Python API.
The full optimization code is thus quite concise.
```python
def optimize(config: OptimizationConfig) -> OptimizationResult:
    solver = pywraplp.Solver.CreateSolver("SCIP")
    supply_to_grid = [
        solver.IntVar(0, 1, "") for _ in range(config.number_of_periods)
    ]  # supplying energy to grid
    charge_from_grid = [
        solver.IntVar(0, 1, "") for _ in range(config.number_of_periods)
    ]  # charging battery from grid
    panels_on = [
        solver.IntVar(0, 1, "") for _ in range(config.number_of_periods)
    ]  # using energy from panels (not dumping it)
    fraction_consumed_from_battery = [
        solver.NumVar(0, 1, "") for _ in range(config.number_of_periods)
    ]  # part of consumption from battery
    consumption_from_battery = []
    ending_battery_level = []
    period_costs = []

    for index in range(config.number_of_periods):
        # constraints
        solver.Add(
            supply_to_grid[index] + charge_from_grid[index] <= 1
        )  # cannot supply and charge at the same time
        solver.Add(
            supply_to_grid[index] + (1 - panels_on[index]) <= 1
        )  # cannot supply and dump energy at the same time
        consumption_from_battery_per_period = (
            config.consumption[index] * fraction_consumed_from_battery[index]
        )
        consumption_from_battery.append(consumption_from_battery_per_period)
        previous_battery_level = previous_value(
            index, ending_battery_level, default=config.initial_battery_level
        )
        ending_battery_level.append(
            previous_battery_level
            - consumption_from_battery_per_period  # consumed from battery
            + config.sun_charging_efficiency
            * config.sun_energy[index]
            * panels_on[index]  # charged from sun
            + config.grid_charging_efficiency
            * charge_from_grid[index]
            * config.max_grid_charge_per_period  # charged from grid
            - supply_to_grid[index]
            * config.max_grid_supply_per_period  # supplied to grid
        )
        solver.Add(
            fraction_consumed_from_battery[index] * config.consumption[index]
            <= previous_battery_level - config.min_battery_level
        )  # we cannot consume more than we have - minimum battery level

        solver.Add(
            ending_battery_level[index] <= config.max_battery_level
        )  # battery level cannot exceed max
        solver.Add(
            ending_battery_level[index] >= config.min_battery_level
        )  # battery level cannot be below min

        # period_costs
        period_costs.append(
            (1 - fraction_consumed_from_battery[index])
            * (config.consumption[index])
            * config.buying_price[index]  # energy consumed from grid (cost)
            + charge_from_grid[index]
            * config.buying_price[index]
            * config.max_grid_charge_per_period  # energy charged from grid (cost)
            - config.selling_price[index]
            * config.supplying_efficiency
            * supply_to_grid[index]
            * config.sun_energy[index]  # energy supplied to grid (revenue)
        )
    # assume that the remaining energy in the battery is sold at the average price
    # (multiplied by -1 because it is a revenue, and we want a cost function)
    remaining_energy_cost = -(
        (ending_battery_level[-1] - config.min_battery_level)
        * config.supplying_efficiency
        * sum(config.selling_price)
        / len(config.selling_price)
    )

    solver.Minimize(solver.Sum(period_costs + [remaining_energy_cost]))

    status = solver.Solve()
    ending_battery_level = [
        EnergyUnit(var.solution_value()) for var in ending_battery_level
    ]
    return OptimizationResult(
        optimum_found=status == pywraplp.Solver.OPTIMAL,
        total_cost=Cost(solver.Objective().Value()),
        supply_to_grid=[
            binary_int_var_to_bool(var.solution_value()) for var in supply_to_grid
        ],
        charge_from_grid=[
            binary_int_var_to_bool(var.solution_value()) for var in charge_from_grid
        ],
        panels_on=[binary_int_var_to_bool(var.solution_value()) for var in panels_on],
        ending_battery_level=ending_battery_level,
        fraction_consumed_from_battery=[
            var.solution_value() for var in fraction_consumed_from_battery
        ],
        period_costs=[Cost(var.solution_value()) for var in period_costs],
        remaining_energy_cost=Cost(remaining_energy_cost.solution_value()),
        min_battery_level_reached=min(ending_battery_level),
        max_battery_level_reached=max(ending_battery_level),
    )
```
The code introduces a few helper variables, but the core logic is what we described above it.
The charging efficiency multipliers allow us to account for energy loss when charging from the grid or solar panels.
The `fraction_consumed_from_battery` variable allows us to distribute our consumption between the battery and the grid.

When we run it, we get the optimal values for switches in each period, but also several other metrics:
```
Configs:
Number of periods: 400
Min battery level: 10.0
Max battery level: 9000.0
Initial battery level: 500.0
Supplying efficiency: 0.9
Sun charging efficiency: 0.8
Grid charging efficiency: 0.9
--------------------
Results:
Remaining energy cost: -2976.749999992534
Min battery level reached: 35.999999999897
Max battery level reached: 581.0
Total cost: -884326.7500000005
Optimum found: True
```
Of course, these results are from dummy data. Getting a cost of -884326 EUR, i.e. a profit of 884326 would be pretty sick.

### Accessing power plant production data

### Weather data

### Predicting power plant production

### Integration and UI