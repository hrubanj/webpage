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
- a word on linear solvers


### Accessing power plant production data

### Weather data

### Predicting power plant production

### Integration and UI