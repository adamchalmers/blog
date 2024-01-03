+++
title = "Free users are incredible"
date = 2024-01-02
description = "Why Cloudflare spends so much on users that spend nothing"
draft = false

[taxonomies]
tags = ["business", "scale", "cloudflare"]
+++

Cloudflare has a really generous free tier, and it gets a _lot_ of free users. I often see comments on Hacker News saying "Cloudflare must be doing something shady, so many people use it for free, and if you're not the customer, then you're the product". This is mostly wrong -- Cloudflare gets a lot of value from its free users, in normal, not-shady ways. Here's why it's helpful to have a ton of free users.

<!-- more -->

# The more customers, the more learning

As a small company, or a small team in a large customer, the most valuable thing you can do is learn. Learn as fast as you can!

Do you learn more from a thousand tiny customers or one big customer? Easy question. One thousand tiny customers will find _way_ more bugs than one big customer. [Cloudflare's API has a _lot_ of endpoints][cf-api], a big customer will maybe use 10% of them. But a thousand free users are all going to need slightly different parts of your API, with slightly different parameters, uploading different kinds of files in different ways. So your API is going to be tested a _lot_ more by 1000 free users than it would be by one big customer.

This is great, because the free users are going to find any weird parts of your API that you forgot about -- any bugs or performance problems. It's like getting free [fuzz testing] or [chaos monkey]. They're going to cause outages and trigger bugs and ensure that all your code paths actually get run in production. If nobody was actually running those code paths, they'd just lie dormant until one day a big customer signs up and starts paying you. Then they start using the old, dusty, untouched code path, and trigger a bug. Wham. Incident occurs, PagerDuty goes off, and the paying customer gets mad. Not a great experience for the paying customer's first day. But if free users were already running that code path, you'd have caught the problem earlier, so by the time your paying customers start using it, it's already battle-tested.

Free users teach you more than just where your bugs are. If your business sells DDOS mitigation, then you want to be constantly mitigating DDOS attempts. You should be attracting as many nasty DDOS attacks as you can, so that you see novel attacks as they're being trialled and learn how to mitigate them. Again, free customers that are always under DDOS help you learn strategies that you can use for high-priority customers.

## Free customers make you scalable

This is _especially_ important because there's some problems you don't run into until you hit significant levels of traffic. If you want to learn how to build scalable software, you want to get as much traffic as possible, so you run into scaling problems and try different approaches to solving them. Hypergrowth is very stressful, but it teaches you a lot. Giving away something valuable for free brings in a lot of traffic. Are those new users paying you? No. But dealing with their traffic teaches you how to scale your software, which is so much more valuable than short-term revenue. When Discord started running [their traffic through Cloudflare][discord] they were really cautious and gave us plenty of advance notice before they turned everything on. They didn't want to bring down Cloudflare when they started shooting terabytes of traffic towards us. But because Cloudflare already knew how to scale those services, it didn't matter. Free users help you scale so that your software stays up when a big customer suddenly starts using your services.

Free users bring in less revenue, but handling them teaches your team a _lot_. If you want to learn as fast as possible -- and you should -- try to get a lot of free users.

# Hobbyists are my favourite users

The most helpful GitHub issues in the world are filed by open-source hackers using a hobby project. If you run a software business, your best-case scenario is that software people file bug reports, because their bug reports are _so helpful_. They're going to include steps to reproduce, their OS details, maybe even packet capture or [HAR files (Chrome traffic replay)][har].

When we had just launched [cloudflared], some guy emailed us with an incredibly detailed bug report about how his server was slowed down when accessed via cloudflared instead of the normal internet. He gave us an example server to reproduce with, tons of logs, even the tracing data showing exactly what cloudflared was doing. He worked with us to reproduce the problem and reran his tests once we had a potential fix. This guy wasn't paying us. He was just a cool engineer who wanted to help us. Free users can be entitled assholes, yes, but many of them understand that they're not paying you, so they can't expect support, so when they file a bug they'd better include as much helpful information as possible, otherwise you won't bother replying. In my experience this kind of really helpful hobby user is way more common than the entitled asshole who thinks you should drop everything and help them.

# Two obvious ones

This concludes the part of my blog where I say something new. But I should cover these other reasons that Cloudflare's free service exists, because they're both real and genuine (I just wanted to talk about the more interesting ones earlier in this post).

## Upsell

[Rita Kozlov][rita] was recently [interviewed on the Hanselminutes podcast][hanselminutes] (transcript available [here][transcript]). He asks her this exact question:

> Scott: I actually have been using Cloudflare for years, but I don't think I've actually ever given you any money. That's an interesting relationship... how is that okay that I've never paid you for anything?
> 
> Rita: [The free tier] is how a lot of people become really familiar with Cloudflare, right? And at the end of the day, we do want you to adopt more of our products and services and build more on us.

Straightforward. Cloudflare's free offerings (e.g. DDOS mitigation and DNS) don't really cost much to operate, so Cloudflare can give it away for free, basically as marketing so that when you need to choose a CDN, load balancer or function-as-a-service provider, you already have warm fuzzy feelings about Cloudflare. I didn't take Business 101, but my impression is that this sort of thing is Business 101. 

This really didn't matter to me as an individual engineer at Cloudflare, because my performance and success in the company wasn't related to how much revenue I sent towards other products. I was rewarded for building stable, fast software that didn't take down our users websites. 

## Altruism

Rita also says that 

> We do actually view people being able to stay online, especially as a commodity, as something that everyone should have access to. So having a free plan is something that's been a part of Cloudflare since the very, very beginning.

In my experience this is something Cloudflare's leadership and employees genuinely believe. The company is willing to spend money to back that belief. But, even if the current leaders were replaced by heartless soulless private equity robots, I still think a very generous free tier is good business sense, for the above reasons.

# Conclusion

Consider offering a very generous free tier for your software. You'll attract a lot of goodwill from users, who'll be willing to write detailed bug reports. The free load testing, fuzzing, and bug reporting that you get from free users will really help battle-test your system before you sign up big paid customers.

Obviously, this is a tradeoff: if part of your system is incredibly expensive to operate (it's very resource-intensive, or requires a lot of time from a dedicated human employee), then the costs of all that free traffic probably outweigh the benefits. But in practice, a lot of software is really cheap to operate. So consider giving it away for free. The more users, the more traffic. The faster you hit scale, the faster you're learning.

[hanselminutes]: https://hanselminutes.com/923/building-a-better-internet-with-cloudflares-rita-kozlov
[rita]: https://twitter.com/ritakozlov_/
[transcript]: https://app.podscribe.ai/episode/92949855
[cf-api]: https://api.cloudflare.com
[fuzz testing]: https://en.wikipedia.org/wiki/Fuzzing
[har]: https://support.google.com/admanager/answer/10358597?hl=en
[chaos monkey]: https://en.wikipedia.org/wiki/Chaos_engineering
[cloudflared]: https://github.com/cloudflare/cloudflared
[discord]: https://www.cloudflare.com/case-studies/discord/