---
layout: post
title: "Building an AI-powered Power Plant - Part 1"
date: 2023-10-17 10:00:00 +0000
categories: [ programming, optimization, electricity, ml, ai ]
---


### Introduction
My parents installed solar panels on their roof. This lets them save electricity costs, and also sell excess electricity to the grid.
Since they are selling for spot prices, choosing when to sell can make a big difference in the profit.
That is an interesting problem to solve, so my brother and I decided to help them with a bit of automation and AI.

The first part of this series will be more of an overview. We won't go too deep into implementation details,
as we'd get buried in them without conveying the main points. In following parts, we'll build upon the initial
solution and dig a bit deeper.

There is one caveat that I should mention. We are not experts on power plants, or electricity trading.
We are learning a lot of the things we are writing about as we are implementing them. So, it is quite likely
that we'll make some incorrect assumptions and we'll have to go back and fix them. We'll try to be as transparent
as possible about this, and we'll try to correct the mistakes as soon as we find them. If you find any, please let us know.


### What are we trying to solve?
We have a power plant that generates electricity when sun is shining. We can use it, charge a battery with it, or send it to the grid.
Using electricity directly or from battery saves us money for buying electricity from the grid. Selling electricity to the grid makes us money.

The following diagram shows the how electricity can flow between the power plant, battery, grid, and consumption.

```mermaid!
flowchart TD
    A[fa:fa-solar-panel Power plant] -->|charges| B[fa:fa-battery Battery]
    A -->|supplies energy for| C(fa:fa-house Consumption fa:fa-computer)
    D[fa:fa-plug Electric grid] -->|charges| B
    B --> |sells to|D
    A --> |sells to|D
    D --> |supplies energy for|C
    B --> |supplies energy for|C
```
When the sun is shining, we can sell electricity to the grid, charge the battery, or consume the electricity.
We can also charge battery from the grid and consume either directly from the grid, or from the battery.
Some combinations of the above make sense, others less so.


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
regulator [OTE](https://www.ote-cr.cz/en/short-term-markets) determines hourly sport prices for the next day.

The prices for the next day are published at 2 PM. This simplifies our problem, because we don't have
to predict them.

### Purchasing vs. selling prices
It is possible to both buy and sell electricity at spot prices. Both prices can then be almost the same–the sell price
is a bit lower because of a transmission fee subtracted from it.

Under our current contract, out buy price is fixed. Our solution must be ready for a situation where it is not.
The buy price will also usually be higher than the sell price. When we switch to spot prices, it will always be higher.

Interestingly, the spot price can be negative. This means that we would have to pay for selling electricity.
It should also mean that we could get paid for consuming electricity, but we need to verify if it actually works this way.

### Default solution
The software from the power plant manufacturer has a default solution for controlling the power plant.
You can, for example, set the maximum battery charge level. When this is reached, the power plant starts selling electricity to the grid.
When the battery is discharged below a certain level, the power plant starts charging the battery.
Own consumption is preferred over both charging and selling. This usually makes sense. Consuming produced electricity
is less lossy than selling it or storing it in the battery. And, as mentioned before, the buy price is usually higher than the sell price.

The default allows for some tuning. But it does not address one important issue:
It does not think ahead. It does not try to optimize when the selling should happen. It might be reasonable to
postpone selling electricity to a peak time when it is expensive.
With the default solution, you have to change settings manually likely multiple times per day.

### Naive heuristic - start without AI
What is the easiest possible solution that can make this better?

In the first iteration, we decided to go with a rule-based scheduler.

In my experience, it is always better to start with the simplest solution, and only improve upon it when the potential benefit outweighs
the cost of improvement. The cost is not just the cost of development, but also the cost of maintenance, which grows with the complexity.
Hence, always start with as little AI as possible, or with no AI at all.

The two main requirements for the scheduler are:
1. you should be able to configure what the power plant does at any given time (prefer charging, prefer selling)
2. it should provide good defaults in case you don't want to configure it for a specific time exactly

Point 1. ensures that you can set power plant behaviour for the 24 hours for which you have spot prices, and
do not have to reset it manually every time the behavior needs to change.

Point 2. should set behavior based on historical data so if you do not want to configure the power plant, it still
does pretty well.

This naive solution leaves out a lot of variables. It does not account for specific weather. It does not predict own consumption.
It does not predict prices beyond the 24 hours fixed by the regulator. It does not care about battery level etc.
Later, we'll see which of these variables are important. Then, we'll add them to the solution.

### Inferring default rules from historical data
Historical data reveal some patters that we would expect.
I think that they react mostly to changes in demand (consumption). Supply is not able to adjust quickly enough to
accommodate different levels of demand during the day. I guess it would be both difficult and expensive to have a power plant run
from 6 AM to 9 AM, and then shut it down for the rest of the day. It is probably cheaper to have it run all day.

Prices are higher in the morning and in the evening, and lower during the day. This is most likely
because in the morning, people are waking up, making breakfast, commuting etc. In the evening, people come home, cook dinner,
watch TV, and so on. During the day, people are at work, so they consume less electricity.
This pattern does not hold on weekends, when people many people are sleeping in, and / or not commuting as much.
The morning peak is significantly smaller on weekends, and the price is generally lower throughout the day.

![]({{ site.baseurl }}/assets/images/electricity_price_by_hour_weekday.png)

*The prices in the images are standardized for year-month combinations.*
The peaks are not so prominent in winter, probably because of heating during the day. They are also shifted more towards
noon.

![]({{ site.baseurl }}/assets/images/electricity_price_by_hour_season.png)

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
On the positive side, we do not have to interact with the power plant directly.

Forcing the user to upload configuration to the server might be too much of a stretch, so we'll use
a rudimentary Telegram-based API for this. 
We create a dedicated conversation for controlling the power plant. The user can upload the configuration
to the conversation and tag the telegram bot. The bot always reads the latest configuration and sets the mode.
If the user wants to deactivate the script, they can send a config message without any config file.

The script also logs the results of its run to the conversation. So, the user will see that the a power plant
mode was either set, or the config was invalid.

```mermaid!
flowchart TD
    A[fa:fa-telegram Telegram chat] -->|downloads config| B[fa:fa-code Cronjob]
    B -->|changes mode| C[fa:fa-solar-panel Power plant API]
    D(fa:fa-user User) -->|uploads config|A
    B --> |logs results|A
```
For additional monitoring, we use a free trial of [Sentry](https://sentry.io/welcome/).

### Results & Lessons learned
It's been a while since I started writing this article. We had a chance to both implement the solution and watch it in action.
I was a resounding failure.

Technically, there was nothing wrong with it. It did exactly what we expected it to do.
The scheduler worked. It used the config correctly to change the power plant mode. It logged the to the Telegram chat.
So what was the problem?

Well, there were several:
1. In the initial solution, I failed to account for the fact that you can charge the battery 
from the grid. I didn't know that it was possible and sometimes even necessary.
When there is no energy from sun for too long, the battery can get discharged below the 10 % threshold. So you need to charge it to prevent degradation.
This is easy to fix. Not a major issue.

2. I didn't double check that my father wants to use a scheduler and that it will be useful for him.
When I presented the idea, he seemed enthusiastic. But I failed to notice that he enjoys fiddling with the
power plant settings. So, the scheduler was not saving him from unpleasant work. This was a major issue.

3. Making non-programmers use JSON configuration non-programmers is a stupid idea. In the spirit of MVP, I wanted to use
the simplest possible configuration format. But JSON seems easy for people who are used to work with
mappings, arrays etc. A simple UI might have been a bit more work, but would probably make the scheduler way more usable.
I am not saying that this was an absolute show stopper. It required a lot of explaining. And the user experience was terrible.
Major issue.

   
### Next steps
Our current solution is a mere scheduler. It was supposed to be an MVP, but it is not really viable.
In the next iteration, we have to pay more attention to user preferences.

The solution can only be viable if it gives user something that they want. We have learned that saving labor
won't be enough.
Its decisions need to generate more profit than the user would be able to generate on their own.
Saving labor might still be worth trying if there is a strong guarantee that the solution will work well without oversight.
The scheduler just executes predefined rules, so it does not react to, e.g., current battery level.

My original plan was to max out the benefits of automation first, and then start adding machine learning.
Based on the learnings above, we'll have to introduce some optimization in the next iteration.

We will, again, start with the simplest possible solution.
Actually, we have already started and the second iteration is well under way.

Stay tuned for the next article!


