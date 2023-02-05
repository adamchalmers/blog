+++
title = "2022 reflections"
date = 2023-02-05
description = "Thinking about my life and work over the last year"
draft = false

[taxonomies]
tags = ["personal", "career"]
+++

Reflecting on the things I liked in 2022. I started writing this in December 2022 and completely forgot to finish it until February next year.

<!-- more -->

# Favourite things in 2022

## Favourite genre

This was my personal year of historical fiction. History was my favourite subject in school, but I've had a hard time learning anything about history since I graduated. This year I decided to really embrace "thousand-page incredibly accurate historical novels" as my primary tool for learning history. I just love the genre. I read [Lonesome Dove] to learn about my new home (Texas); [Pachinko], [Shogun] and its followup [Tai-Pan] to learn a bit about Asian history, and continued my reread of Colleen McCullough's [Masters of Rome] series, which tells the story of how the Roman republic collapsed and became the Roman Empire. My high school teachers changed my life by giving me a worn copy of [The First Man in Rome] in high school, and during the 2020 pandemic I decided to start rereading them all. If you've got any recommendations for good in-depth historical novels, please let me know!

On the TV, I loved [For All Mankind] and [Outlander], set in an alternate 20th century and the real 18th century respectively. 

## Favourite movie

I think my favourite movie I saw in theatres was Everything Everywhere All At Once -- I laughed, I cried, mostly I laughed. Barbarian is a close runner-up for those who like horror.

## Favourite music

Concerts attended: Chris Thile, Sarah Jarosz, Bon Iver, Covet, Metric, Bela Fleck and the Punch Brothers

Favourite musician: Sarah Jarosz. She was my favourite live concert of the year -- we saw her in a small Austin church with _phenomenal_ sound quality.

Song most repeated: [Crazy On You]. Why had nobody shown me this song before? It's perfect!

## Favourite cooking

I got really into cooking Chinese-style noodle dishes thanks to [Wil Yeung](https://www.youtube.com/watch?v=DOKvDbkJrdE). They've become a staple of our weekday cooking. Yeung's been my favourite YouTube cooking channel for a few years now. His videos are so calming, and I really appreciate his explanations of various Asian ingredients I'm not familiar with.

# Blogging

My goal for this blog has always been: build up a collection of useful guides to common programming problems, especially where no good guide exists. I really feel like 2022 was the year I achieved this goal. I've seen library maintainers linking my articles in the Tokio discord to help explain a point, or coworkers linking my guides to other coworkers. My philosophy to blogging has always been "if I'm confused by this, someone else is probably confused by this too. If I'm searching for an article, someone else is probably searching for an article too." I think this year I definitely proved that to myself.

I was also really happy with my [Kubernetes article], which was my first time blogging about something other than Rust. It did very well on Hacker News and some coworkers who maintain our Kubernetes clusters liked it. I found k8s so confusing when at first, so I'm glad I can help flatten the difficulty curve for others.

# Career

In January of 2022 I helped start a [new team at Cloudflare, Data Loss Prevention (DLP)][dlp-blog]. It was just one manager, one product manager, and one programmer (me). In the next 12 months we designed and built a new data proxying service that can scan traffic and report which regexes, filetypes, virus signatures etc are being transmitted. This went well, and several other teams started running their traffic through our service to add features to their products. By the end of the year, we had successfully closed contracts, gotten real customers using it, onboarded several other teams who wanted us to scan their data, and expanded the team from 1 engineer to 8. It's been phenomenal getting to build a new product from scratch, especially a product that people actually want to use and pay for.

At the start of the year I was talking to a good friend who's further along in his career -- he was actually my manager for several years. His advice was that my current biggest weakness as a programmer was my lack of experience with lower-level network protocols and Linux internals. I agree -- at this point, writing code is relatively straightforward. My biggest challenge is figuring out _what_ code to write to interface with the computer. I tried to improve on these things over the last 12 months, but honestly, shipping DLP took up most of my time. Luckily at Cloudflare we have a really good technical infrastructure to build on, which makes shipping new products easy. But it did mean I was isolated from a lot of low-level OS details. Overall I improved on these weaknesses this year, but not as much as I'd like. Hopefully 2023 will see more progress in that direction.

# Most popular tweet

Unsurprisingly, my fake rust tweets were my top tweets this year. This one hit around 1.7k likes.

[![Tweet from my personal Twitter, changed to impersonate the official Rust twitter. The tweet says "We're adding a fifth string type called "Strang". It's subtly different to String but we're not going to tell you how."](/fake-rust-twitter/strang.png)][strang]

# Personal

Jordan and I had our wedding this year. We got married on paper in 2020, because the coronavirus pandemic made it very unlikely that the Australian and American family would be in the same country anytime soon. Being married continues to be a huge psychological shift that surprises and satisfies me. I hope that by the time I write my end-of-year 2023 blog post, I'll be talking about pregnancy and preparing for children.

[The First Man in Rome]: https://www.goodreads.com/book/show/480570.The_First_Man_in_Rome
[Masters of Rome]: https://en.wikipedia.org/wiki/Masters_of_Rome
[Shogun]: https://en.m.wikipedia.org/wiki/Sh%C5%8Dgun_(novel)
[Tai-Pan]: https://en.wikipedia.org/wiki/Tai-Pan_(novel)
[Lonesome Dove]: https://en.wikipedia.org/wiki/Lonesome_Dove
[Pachinko]: https://en.wikipedia.org/wiki/Pachinko_(novel)
[For All Mankind]: https://en.wikipedia.org/wiki/For_All_Mankind_(TV_series)
[Outlander]: https://en.wikipedia.org/wiki/Outlander_(TV_series)
[Crazy On You]: https://www.youtube.com/watch?v=OZuW6BH_Vak
[Kubernetes article]: https://blog.adamchalmers.com/kubernetes-problems
[dlp-blog]: https://blog.cloudflare.com/inline-data-loss-prevention/
[strang]: https://twitter.com/adam_chal/status/1590721811876306944
