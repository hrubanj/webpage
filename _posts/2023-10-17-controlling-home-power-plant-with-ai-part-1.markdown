---
layout: post
title: "Controlling home power plant with AI - Part 1
date: 2023-10-17 10:00:00 +0000
categories: [ programming, optimization, electricity, ml, ai ]
---

# Outline
- problem statement
- the big picture
- variables under control
- spot prices
- purchasing vs. selling prices
- default solution
- naive heuristic - start without AI
- historical data - price trends and defaults
- own consumption
- results
- adding intelligence - what can be improved
- other parts
  - integration with software, deployment, technical monitoring
  - drilldown into variables prediction and optimization
  - business evaluation
### Introduction

### What are we trying to solve?
We have a power plant that generates electricity when sun is shining. We can use it, charge a battery with it, or send it to the grid.
Using electricity directly or from battery saves us money for buying electricity from the grid. Selling electricity to the grid makes us money.

### The big picture

### Variables under control
Own consumption always takes precedence regardless of our settings. This is actually economically rational, since
it should reduce losses due to charging / transmission / storing / discharging. Also buy price for electricity
is usually higher than sell price.

We can set the power plant to prefer charging the battery, or selling to the grid, or just throwing electricity away.
Throwing electricity away makes sense only if the battery is full and sell price is negative.

The battery should not be discharged below 10 % of its capacity, and should not be charged above 90 % of its capacity, otherwise
it will degrade faster.

### Spot prices
We sell electricity to the grid at spot prices. Spot prices are hourly prices for electricity on the market. In the Czech Republic, the
regulator (OTE)[https://www.ote-cr.cz/en/short-term-markets] determines hourly sport prices for the next day.

The prices for the next day are published at 2 PM. This simplifies our problem, because we don't have
to predict them.

### Purchasing vs. selling prices
It is theoretically possible to both buy and sell electricity at spot prices. We decided not to go through
the hassle at first, so, we did not change the contract for buying electricity.
The buying price is almost always higher than the selling price, so it makes sense to use the electricity we generate directly.

### Default solution
The software from the power plant manufacturer has a default solution for controlling the power plant.
You can set the maximum battery charge level. When this is reached, the power plant starts selling electricity to the grid.
When the battery is discharged below a certain level, the power plant starts charging the battery.
Own consumption is preferred over both charging and selling. This makes sense because consuming produced
electricity will almost always be cheaper than buying it from the grid (as mentioned in the previous section).

This is a reasonable default that allows for some tuning. But it does not address one important issue:
It does not think ahead. It does not try to optimize when the selling should happen. It might be reasonable to
postpone selling electricity to a peak time when it is expensive, and with the default solution, you have to do this manually.

### Naive heuristic - start without AI
What is the easiest possible solution that might improve upon the default behavior?

Actually, a simple scheduler might do the job.

In my experience, it is always better to start with the simplest solution, and only improve upon it when the potential benefit outweighs
the cost of improvement. The cost is not just the cost of development, but also the cost of maintenance, which grows with the complexity.
Hence, always start with as little AI as possible, or with no AI at all.

So, yes, we decided to implement a simple configurable scheduler. A bit anti-climactic, I know.

The two main requirements for the scheduler are:
1. you should be able to configure what the power plant does at any given time (prefer charging, prefer selling)
2. it should provide good defaults in case you don't want to configure it for a specific time exactly

Point 1. ensures that you can set power plant behaviour for the 24 hours for which you have spot prices, and
do not have to reset it manually every time the behavior needs to change.

Point 2. should set behavior based on historical data so if you do not want to configure the power plant, it still
does pretty well.

This naive solution leaves out a lot of variables–it does not account for specific weather, it does not predict own consumption,
it does not predict prices beyond the 24 hours fixed by the regulator, it does not care about battery level etc.
Many things can be important, but having a simple default solution will allow us to better gauge the benefit of adding
more intelligence.

### Historical data - price trends and defaults

