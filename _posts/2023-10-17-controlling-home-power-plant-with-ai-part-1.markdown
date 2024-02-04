---
layout: post
title: "Building an AI-powered Power Plant - Part 1"
date: 2023-10-17 10:00:00 +0000
categories: [ programming, optimization, electricity, ml, ai ]
---
<link href="//maxcdn.bootstrapcdn.com/font-awesome/4.2.0/css/font-awesome.min.css" rel="stylesheet">
<a href="{{ site.baseurl }}/index.html"><i class='fa fa-home'></i> Home</a>

### Introduction
My parents installed solar panels on their roof. Besides using generated electricity, they can sell the excess to the grid.
Choosing when to sell influences how much profit you make, because electricity prices fluctuate. 
My brother and I thought it was a nice optimization problem. So, we decided to take a stab at it.

This is a first post in a series. Here, we'll make an overview of the problem. We'll also sketch up the first iteration of the solution.
In following posts, we'll learn from its shortcomings, and improve upon it.

There is one caveat that I should mention. We are not experts on power plants, or electricity trading.
We are learning a lot of the things as we are implementing them. So, it is quite likely
that we'll make some incorrect assumptions, and we'll have to go back and fix them. I'll try to be as transparent
as possible about this, and I'll try to correct the mistakes as soon as we find them. If you find any, please let me know.


### What are we trying to solve?
The solar panels generate electricity when sun shines upon them.

The following diagram shows how electricity can flow between the power plant, battery, grid, and consumption.

```mermaid!
flowchart TD
    A[fab:fa-solar-panel Power plant] -->|charges| B[fab:fa-battery Battery]
    A -->|supplies energy for| C(fab:fa-house Consumption fab:fa-computer)
    D[fab:fa-plug Electric grid] -->|charges| B
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
But it can grow almost arbitrarily complex. We can refine our predictions of weather, consumption, prices, effectiveness
of charging and so on.

We need to be strict about what we want to achieve. Rather than building a spaceship that's never finished, we want a serviceable bike
that can be quickly tested and improved. (And eventually, turned into a spaceship if it brings enough value).
From the start, we'll use the least amount of effort
to create somewhat functional solution, and then we'll improve it in iterations. We'll only improve 
the parts that seem to bring the most benefit at the time.

We'll be making a lot of bold assumptions, and using some atrocious implementation shortcuts. 

Additionally, there won't by much AI in the first iteration. I'll explain why in the following sections.

### Variables under control

We can set the power plant to prefer charging the battery, selling to the grid, or just dumping electricity.
We can also set it to charge the battery from the grid.
Dumping excess solar electricity makes sense if the battery is full and sell price is negative.

The battery should not be discharged below 10 % of its capacity, and should not be charged above 90 % of its capacity, otherwise
it will degrade faster. We'll take these thresholds as hard constraints.

### Spot prices
We sell electricity to the grid at spot prices, determined for hourly intervals by the market regulator ([OTE](https://www.ote-cr.cz/en/short-term-markets)).

The prices for the next day are published at 2 PM. This simplifies our problem, because we don't have
to predict them.

### Purchasing vs. selling prices
The buy price will also usually be higher than the sell price.
It is possible to both buy and sell electricity at spot prices. In this case, the sell price
is still a bit lower because of a transmission fee subtracted from it.

Even though, the buy price is fixed under out current contract, we must be ready to work with both spot buy and spot sell prices.

Interestingly, the spot price can be negative. This means that we would have to pay for selling electricity.
It should also mean that we could get paid for consuming electricity, but we need to verify if it actually works this way.

### Default solution
The software from the power plant manufacturer has a default solution for controlling the power plant.
You can, for example, set the maximum battery charge level. When it is reached, the power plant starts selling electricity to the grid.
When the battery is discharged below a certain level, the power plant starts charging the battery.
Own consumption is preferred over both charging and selling. This usually makes sense. Consuming produced electricity
is less lossy than selling it or storing it in the battery. And, as mentioned before, the buy price is usually higher than the sell price.

The default allows for some tuning. But it does not address one important issue:
It does not think ahead. It does not try to optimize when the selling should happen. It might be reasonable to
sell the electricity at a price peak time even if the battery is not full.
With the default solution, you have to change settings manually, likely multiple times per day.

### Naive heuristic - start without AI
What is the easiest possible solution that can make this better?

In the first iteration, we decided to go with a rule-based scheduler. If you start with something complex, you often
spend a lot of time on solving problems that you don't even have.

The two main requirements for the scheduler are:
1. you should be able to configure what the power plant does at any given time (prefer charging, prefer selling, charge from grid)
2. it should provide good defaults in case you don't want to configure it for a specific time exactly

Point 1. ensures that you can set power plant behaviour for the 24 hours for which you have spot prices, and
do not have to reset it manually every time the behavior needs to change.

Point 2. should set behavior based on historical data so if you do not want to configure the power plant, it still
does pretty well.

The scheduler's decision-making can then look something like this:

| Hour starting         	 | 7            	 | 8            	 | 9                 	 | 10           	 |
|-------------------------|----------------|----------------|---------------------|----------------|
| Global default        	 | CHARGING_SUN 	 | CHARGING_SUN 	 | CHARGING_SUN      	 | CHARGING_SUN 	 |
| Weekday settings      	 | SELLING_SUN  	 | -     	        | SELLING_SUN       	 | -    	         |
| Day-specific settings 	 | -     	        | -     	        | CHARGING_SUN_GRID 	 | -     	        |
| <b>Resulting mode</b> 	 | SELLING_SUN  	 | CHARGING_SUN 	 | CHARGING_SUN_GRID 	 | CHARGING_SUN 	 |

<span style="font-size:0.5em;">CHARGING_SUN sets the system to prefer charging over supplying, SELLING_SUN sets it to prefer supplying, CHARGING_SUN_GRID sets it to charge from both sun and grid. These are just example modes. You can come up with more possibilities.</span>

The scheduler always uses the most time-specific settings that it can find.
You set global default and weekday-specific settings based on patterns in historical data. You can then override these settings for specific days, e.g.
when the day is unusually sunny, or when electricity prices are unusually high.

We need to combine these modes with reasonable settings in the manufacturer's UI that will, for example, ensure that the
battery is not discharged below 10 % not matter what mode the scheduler sets.


### Inferring default rules from historical data
Most patterns we see in historical data should not surprise us.
I think that electricity prices react mostly to changes in demand (consumption). Supply is not able to adjust quickly enough to
accommodate different levels of demand during the day. I guess it would be both difficult and expensive to have a power plant run
from 6 AM to 9 AM, and then shut it down for the rest of the day. It is probably cheaper to have it run all day.

Prices are higher in the morning and in the evening, and lower during the day. This is probably
because, in the morning, people are waking up, making breakfast, commuting etc. In the evening, people come home, cook dinner,
watch TV, and so on. During the day, people are at work, so they consume less electricity.
This pattern does not hold on weekends, when people many people are sleeping in, and / or not commuting as much.
The morning peak is significantly smaller on weekends, and the price is generally lower throughout the day.

![]({{ site.baseurl }}/assets/images/electricity_price_by_hour_weekday.png)

*The prices in the images are standardized for year-month combinations (I subtract mean price for year-month for each value and divide by the standard deviation for that year-month).*
The peaks are not so prominent in winter, probably because of heating during the day. They are also shifted more towards
noon.

![]({{ site.baseurl }}/assets/images/electricity_price_by_hour_season.png)

This shows us that we should expect high prices from 6 AM to 9 AM, and from 5 PM to 9 PM, except
for weekend mornings, and that we should calibrate our assumptions for winter and summer.


### Implementation
We keep the spirit of simplicity also for the actual implementation.

The solution only needs to be able to read a configuration file and set the power plant mode.
We use a hierarchical configuration outlined above. Each entry has a start time, end time, and a mode. Entries
can also have specific date, weekdays, or none of this. Date-specific entries take precedence over
weekday-specific entries, which take precedence over entries without any of these. If no entry is
found for given time, the default mode is used.
This lets users define exact mode based on assumed price and consumption, but lets us also define
defaults that should function reasonably well if the user does not want to configure the power plant too often.

This means that we need a script that regularly checks the configuration file and sets the mode.
We use a cron job for this, and run it on a cheap VPS.

#### Communication protocol

The power plant manufacturer does not provide any public API for setting the power plant mode.
Hence, we had to reverse-engineer it from their UI.

On the positive side, we can use HTTP requests to call the manufacturer's server, and
we do not have to interact with the power plant directly via some exotic protocol. But that is probably
the only convenient part about it.

My brother had to become a bit of a cryptographer to figure out what the UI does. Just to give you an idea:
 - The API is not documented.
 - You can't rely on the status code to tell you if the request was successful. It always returns 200 (success) status code. 
 - The response body is a list of numbers. Each number corresponds to a setting. You have to figure out which number is which setting based on their order.
 - The response body contains some Chinese characters. Translating them can give you a hint about what the response means.

And obviously, since the API is non-public, it can change any time. Before we managed to finish the implementation, there was at least
one braking change.

The reverse-engineering process is fascinating and it would probably deserve its own article. I'll try to convince my brother to write it.

#### UI

Forcing the user to upload configuration to the server might be too much of a stretch, so we use
a rudimentary Telegram-based API for this. 
We create a dedicated conversation for controlling the power plant. The user can upload the configuration
to the conversation and tag the telegram bot. The bot always reads the latest configuration and sets the mode.
If the user wants to deactivate the script, they can send a config message without any config file.

#### Observability

The script also logs the results of its run to the conversation. So, the user will, for example, see that the power plant
mode was either set, or the config was invalid.

```mermaid!
flowchart TD
    A[fab:fa-telegram Telegram chat] -->|downloads config| B[fab:fa-code Cronjob]
    B -->|changes mode| C[fab:fa-solar-panel Power plant API]
    D(fab:fa-user User) -->|uploads config|A
    B --> |logs results|A
```

### Results & lessons learned
It's been a while since I started writing this article. We had a chance to both implement the solution and watch it in action.
The result was a bit of a disappointment.

It did exactly what we expected it to do.
The scheduler worked. It used the config correctly to change the power plant mode. It logged the to the Telegram chat.
So what was the problem?

Well, there were several:
1. In the initial solution, I failed to account for the fact that you can charge the battery 
from the grid. I didn't know that it was possible and sometimes even necessary.
When there is no energy from sun for too long, the battery can get discharged below the 10 % threshold. So you need to charge it to prevent degradation.
This is easy to fix. Not a major issue. (I've already fixed this in the above text to avoid confusion.)

2. I didn't double-check that the intended user, i.e. my father, wants to use a scheduler and that it will be useful for him.
When I presented the idea, he seemed enthusiastic. But I failed to notice that he enjoyed fiddling with the
power plant settings. So, the scheduler was not saving him unpleasant work. It was rather robbing him of this hands-on experience.
This was a major issue.

3. Making non-programmers use JSON configuration is a stupid idea. In the spirit of MVP, I wanted to use
the simplest possible configuration format. But JSON seems easy for people who are used to work with
mappings, arrays etc. A simple UI might have been a bit more work, but would probably make the scheduler way more usable.
I am not saying that this was an absolute show stopper. It required a lot of explaining. And the user experience was terrible.
Major issue.

   
### Next steps
The scheduler was not as successful as I hoped. It isn't even and MVP. It is not viable.
But it did give us some valuable insights. In the next iteration, we have to pay more attention to user preferences.

The solution can only be viable if it gives user something that they want. We have learned that saving labor
won't be enough.
Its decisions need to generate more profit than the user would be able to generate on their own.
Saving labor might still be worth trying if there is a strong guarantee that the solution will work well without oversight.
The scheduler just executes predefined rules, so, it does not react to, e.g., current battery level.

My original plan was to max out the benefits of automation first, and then start adding machine learning.
Based on the learnings above, we'll have to invert this order.

We are already working on the second iteration. Stay tuned for the next article!
