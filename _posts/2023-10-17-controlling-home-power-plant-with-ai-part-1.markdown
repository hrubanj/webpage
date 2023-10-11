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
regulator determines hourly sport prices for the next day. The prices are published at 14:00 for the next day.
TODO: add regulator name and link to the website

### Purchasing vs. selling prices
