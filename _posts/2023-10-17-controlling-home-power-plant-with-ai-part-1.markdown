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
My parents installed solar panels on their roof. This lets them save electricity costs, and also sell excess electricity to the grid.
Since they are selling for spot prices, choosing when to sell can make a big difference in the profit.
That is an interesting problem to solve, so my brother and I decided to help them with a bit of automation and AI.

The first part of this series will be more of an overview. We won't go too deep into implementation details,
as it might get too long without conveying the main points. In following parts, we'll build upon the initial
solution and dig a bit deeper.


### What are we trying to solve?
We have a power plant that generates electricity when sun is shining. We can use it, charge a battery with it, or send it to the grid.
Using electricity directly or from battery saves us money for buying electricity from the grid. Selling electricity to the grid makes us money.

[mermaid diagram -> power plant, battery, grid, consumption, sell, buy]

### The big picture
On the face of it, this problem is really simple. We are just deciding whether to sell or store electricity.
But it can grow almost arbitrarily complex–we can refine our predictions of weather consumption, prices, effectiveness
of charging and so on.

We need to be strict about what we want to achieve, or we'll end up in analysis paralysis.
From the start, we'll be aggressively pursuing the Pareto principle. We'll use the least amount of effort
to create somewhat functional solution, and then we'll improve it in iterations. We'll only improve 
the parts that seem to bring the most benefit at the time.

You might be cringing a bit when reading the following sections–we'll be making a lot of bold assumptions,
and using some atrocious implementation shortcuts. But that's ok. It will allow us to see what is important
and where the basic ugly solution actually works well enough.

Yes, this also means that there won't by much AI in the first iteration. I'll explain why in the following sections.
Trust me, it will make sense. If you stick around for the next parts, we'll give you all the AI you could hope for.

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
Looking into the historical data reveals some patters that we would expect.
Electricity prices are higher in the morning and in the evening, and lower during the day. This is most likely
because in the morning waking up, making breakfast, commuting etc. In the evening, people come home, cook dinner,
watch TV, and so on. During the day, people are at work, so they consume less electricity.
This pattern does not hold on weekends, when people many people are sleeping in, and / or not commuting.
The morning peak is significantly smaller on weekends, and the price is generally lower throughout the day.
[picture 1]

The peaks are not so prominent in winter, probably because of heating during the day.
[picture 2, january vs. july]

This shows us that we should expect high prices from 6 AM to 9 AM, and from 5 PM to 9 PM on weekdays, except
for weekend mornings, and that we should calibrate our assumptions for winter and summer.

This gives us a good starting configuration.

### Implementation
We keep the spirit of simplicity also for the actual implementation.

The solution only needs to be able to read a configuration file and set the power plant mode based
on configuration setting for current time.

We'll use a hierarchical configuration. Each entry has a start time, end time, and a mode. Entries
can also have specific date, weekdays, or none of this. Date-specific entries take precedence over
weekday-specific entries, which take precedence over entries without any of these. If no entry is
found for given time, the default mode is used.
This lets users define exact mode based on assumed price and consumption, but lets us also define
defaults that should function reasonably well if the user does not want to configure the power plant too often.

This means that we need a script that regularly checks the configuration file and sets the mode.
We'll use a cron job for this and run it on a cheap VPS.

Setting the power plant mode is a bit tricky, because the manufacturer does not provide any public API for that.
Hence, we need to figure out what endpoints are called from the manufacturer's UI and replicate that.
This is also a potential point of failure, because the manufacturer might change the UI and break our script.

Forcing the user to upload configuration to the server might be too much of a stretch, so we'll use
a rudimentary Telegram-based API for this. 
We create a dedicated conversation for controlling the power plant. The user can upload the configuration
to the conversation and tag the telegram bot. The bot always reads the latest configuration and sets the mode.
If the user wants to deactivate the script, they can send a config message without any config file.

The script also logs the results of its run to the conversation. So, the user will see that the a power plant
mode was either set, or the config was invalid.

For additional monitoring, we use a free trial of Sentry.

### Results
This initial solution should mainly save user's time. We did not expect that it would outperform
user configuring the power plant manually, because it only carries out explicit user configuration.
What we can do now and how well it serves us.
- how user is able to work with it

### Making it intelligent
Our current solution is basically a scheduler.
It requires manual input to function reasonably well.
Someone should configure it every day, otherwise it will decide based on rough approximations.
This is still better than having to switch the power plant manually every hour, but there is room for improvement.

There are two main ways in which we can improve the solution–adding automation, and making it more user friendly.

We will almost certainly go for the automation first. It should not only be more fun to implement,
but it will also reduce the need for manual input, and thus should be more profitable.
I have already sketched out a few variables that the optimization will account for in the previous sections.
Keeping the spirit of our Pareto approach, we will start with a simple solution that uses a lot of heuristics,
and we will start by replacing those that seem to bring the most improvement.

As for user-friendliness, the UI is quite terrible for a non-technical user. Producing valid JSON files can be a bit complicated
if you have no experience with them. There is no validation, so you have to wait if the script will work or not.
We will not focus on the UI in the next iteration too much, but we might do some small quality of life tweaks.
If all goes well, we will create a proper UI in the next stage.

So, stay tuned for the next part! And look forward to some actual machine learning.


