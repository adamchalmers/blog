+++
title = "Rustconf 2023 recap"
date = 2023-09-24
description = "What it's like in-person"

[taxonomies]
tags = ["rust", "in-person", "conferences"]
+++

I got back from Rustconf 2023 last week, and I had a great time! I thought I'd write up my impressions of the conference, for anyone curious about what it's actually like.

<!-- more -->

# Talks

There were a lot of really interesting talks! You'll be able to watch them all online soon. Some of my favourites were:

 - [Yoshua Wuyts](https://yoshuawuyts.com/) talking about effect systems. The problem: if you have a function like "copy", how do you make a normal copy, an async copy, a fallible copy (which returns Result), a fallible async copy, a const copy, etc? You don't want to have _n_ "flavours" of function which can be combined to make 2<sup>n</sup> different functions. I don't know if Wuyts has the right solution, but I really appreciated the systematic explanation of the problem itself and why it's worth taking seriously. 
 - [Rain](https://sunshowers.io/) gave a talk about the cursed parts of Unix signals, which have caused me pain before (see [Signals vs. Servers](/signals-vs-servers))
 - [Nikolai Vazquez](https://twitter.com/NikolaiVazquez) talked about `const` -- I love writing compile-time code, because it pushes runtime errors into compile errors. So I'm always looking for ways to do more in `const` functions.
 - [Lisa Crossman](https://twitter.com/Lisa_Crossman) gave a fascinating talk about bioinformatics, and how she's using Rust to speed up genomic analysis and other big data biology problems. I learned so much talking to her after the conference, about how different various kinds of plants and animals are at a genetic level -- some wheat species have six sets of chromosomes! That's crazy!
 - [Jan David Nose](https://www.jdno.dev/) talked about Rust's infra (distributing static Rust toolchains, compilers) etc. As the number of Rust programmers grows, the Rust project's bandwidth bills are growing exponentially to keep up with everyone's Rust downloads. Currently, companies sponsoring the Rust Foundation cover this bill, but the Foundation is trying to reduce their total egress so that they don't require exponentially-growing generosity from sponsors to keep hosting Rust.
 - [Will Crichton](https://willcrichton.net/) spoke about adding interactive quizzes into the Rust book, which lets readers test their understanding, and lets the Rust project evaluate what new Rust programmers have the greatest difficulty learning. It's a really fascinating approach to education! Imagine being able to A/B test different ways of explaining a concept like "lifetimes" or "Pin pointers", by comparing how many readers got quiz questions correct before and after a change. Really fascinating.

There were lots of really great talks, but I couldn't go to all of them (the conference was dual-track, i.e. sometimes two talks were scheduled in different rooms simultaneously). Sometimes a talk was too advanced for me, or too basic for me, but I felt like they would be right for someone else. Some talks I really wanted to attend, but my brain was too tired/distracted in the moment to pay proper attention. So I'm definitely going to go back and rewatch several of the talks when they're online. 

Last year, I watched some recorded Rustconf talks with my friend [Jyn](https://twitter.com/jynelson514) once they were all released online. I think that's a great idea -- get some other Rust programmers together, choose a few videos and really watch them paying close attention. You can pause them and go over each slide to make sure you really understand a new concept. This was the only way I could understand advanced talks from last year, like [the one about "C++ move constructors"](https://www.youtube.com/watch?v=UrDhMWISR3w) -- slowing it down and discussing it with friends. Strongly recommended way to learn hard material.

# People

I really enjoyed the social part of Rustconf -- I'm probably two standard deviations more extroverted than the average programmer so that is to be expected. It was great to meet some people I'd only known as usernames, like [Inanna](https://recursion.wtf), [Esteban](https://hachyderm.io/@ekuber) and [Florian](https://hachyderm.io/@skade). And I bumped into some former coworkers, including [Martin Pool](https://github.com/sourcefrog), who I haven't seen in ten years! When I had my first-ever tech internship, I was assigned to his team. We've both sinced moved from Sydney to the USA, so it was great to see him again.

The last time I attended Rustconf was 2019, when Cloudflare sponsored it. I mostly hung out with Cloudflare coworkers then. This year I didn't know anyone else going. So I met a lot of strangers! 

A few years ago I went to Strange Loop, an awesome general-purpose programming conference. I found it much easier to talk to strangers at Rustconf than Strange Loop, because Rust programming is more of a niche. My go-to question when I met someone new was "so, what do you do with Rust?" and that's usually enough to start some interesting conversation! Finding shared passions is easy because we're all at a Rust conference, after all. So, even if you're a bit shy, I think Rustconf was pretty easy to socialize at. 

Rustconf had a pretty broad range of attendees -- everyone from senior Rust project members to university undergraduate students. One of the best things you can do at a conference is just meet a lot of people and help them find the other people they should meet. If there's a student looking for an internship, introduce them to someone looking for an intern! Helping people find each other was really great. I really enjoyed meeting a whole bunch of different people and getting to know them.

At a conference, there's always people who know each other, and are going to make breakfast plans with their friends. That's great! For me, I didn't know many people there. So I resolved to organize casual breakfasts every day. This was really easy -- I just posted in the conference Discord, "I'm going to have breakfast at _place_ around _time_ -- please feel free to come along too!" Everyone wants someone to make the casual, open, welcome plans. If nobody has yet, then congratulations -- you can be the person to make them! You don't require any permission from the conference, or to be an official spokesperson or a big community member. Just choose a time and place for other people who don't have plans, to all have plans together. I think this worked well and helped people feel comfortable. Strongly recommend doing this at conferences.

I'm glad I talked to so many people, because I had a great time and made some friends I would never have met otherwise, like Zach, a student on the opposite side of the country. But we started talking about his work (improving CPU cache by letting the compiler do static analysis on where variables are used) and my work on optimizing KittyCAD's latency, and just had a great time! I used to find talking to strangers at conferences stressful, but apparently now I love it.

I use social media and read blogs a lot, so I've been following various Rust maintainers and contributors online for a long time. It was really cool to get to meet some of them in real life. If you go to conferences, you'll frequently meet people whose work you're familiar with, but who won't know you at all. Some people find this pretty awkward, and avoid talking to them for that reason. Here's a suggestion: people love talking about their passions. And if somebody is passionate enough about their software or side-project to work on it in their spare time, unpaid, then they're probably more than happy to talk about it! So, my advice: people usually enjoy talking about their passions, and enjoy answering your questions about them. It was great to be able to ask e.g. Rain all my questions about developing [nextest](https://nexte.st/), or [Ed Page](https://epage.github.io/) questions about [winnow] and [clap]. At your next conference, ask people about their passion projects. You'll probably learn a lot, and have some really fun conversations. Although remember to factor in the typical level of programmer introversion :) I'm very extroverted, but even I was tired of talking to people by the end of dinner each night.

# The Unconference

When I bought my Rustconf ticket, the website asked me if I wanted to register for "The Unconference". I didn't really know what that entailed, so I just said "yes" -- the more Rust the better, right? 

I went in knowing nothing about it, but Unconference was my favourite part of the trip. There were no planned sessions. In the morning, we all met in the one available room, and people proposed session topics on post-it notes. Then everyone voted on which topics they wanted to talk about. Unconference facilitators put together a schedule, and we divided the room into four corners. Every hour, 4 different topics got discussed, one in each corner. 

The discussions were really interesting. Nobody had prepared a talk, they were just round-table discussions. Have something to contribue? You put up your hand, and get a chance to say it. Topics I went to included:

 - Sustainable Rust open-source development
 - Advanced linting (better built-in lints, and better custom, user- or project-specific lints)
 - Rust education (mostly in universities, but elsewhere too)
 - Improving Cargo for day-to-day use
 - Problems and improvements with "system crates" (crates link to non-Rust code, like [openssl-sys](https://lib.rs/crates/openssl-sys))

I learned a _lot_ from these sessions! In some ways, the Unconference wasn't particularly aimed at people like me. Most of the topics were aimed at people who are actively working on Rust or its ecosystem. I mostly write Rust for private companies, and while I open source what I can, nothing on my GitHub is anywhere near close enough to be part of "the Rust ecosystem". Nonetheless it was really fascinating hearing from Rust project members, like [Manish](https://twitter.com/ManishEarth) and [Ed](https://hachyderm.io/@epage), about what Rust contributors think about and what problems they're trying to solve, usually things that I would never have thought about in my day-to-day as a Rust user. [Predrag](https://twitter.com/PredragGruevski) of [cargo semver-checks](https://crates.io/crates/cargo-semver-checks) had a lot of really interesting thoughts about the wider Rust ecosystem -- I really appreciated his perspective. Because I didn't have that much to say, I tried to make myself useful by taking very detailed notes at each session.

# Albuquerque

I love New Mexico. I wish I'd planned out a full week + weekend in New Mexico, so I could have driven out to some of its beautiful nature and hiking (the American Southwest is one of the most gorgeous places I've ever been). There were parts of downtown Albuquerque I really liked -- they had great coffee shops, and some really good restaurants (Farina Pizzeria, Roma Bakery, Suenos Coffee). 

I feel very awkward about typing the next paragraph, but a few people have asked me about this aspect, so I am going to try to be honest. Walking around downtown Albuquerque was not very pleasant. Google Maps kept routing me to walking-distance cafes, pharmacies, etc via routes that took me through large groups of homeless people. Downtown Albuquerque had basically zero pedestrian foot traffic, which made it feel less safe than walking past a lot of homeless people in a busy city like NYC or SF. Nothing bad happened to me on the trip -- one homeless person yelled at me while I was walking home alone after dinner, pretty minor incident -- but I definitely felt tense and started planning more roundabout routes to avoid corners with a lot of homeless people. I'm a 30-year-old, six-foot tall, gender-conforming man, so honestly I don't usually take my safety into consideration when making nighttime plans. But even I felt like I should try to avoid walking alone at night and stick to groups of conference-goers walking back to the same hotels. I don't know how other conference attendees who were smaller, more feminine or visibly queer felt -- I can't speak for them. But I know I was not the only tall man who chose his walking routes carefully to minimize his risks.

Maybe I'm an outlier. I am definitely unusually jumpy about large groups of homeless people after a bad incident in Austin last year. A lot of my discomfort was due to being new to Albuquerque -- I don't know whether it's actually a bad idea to walk around downtown Albuquerque at night, or if it just _felt_ like one. If Rustconf returns to Albuquerque next year, I'll get a hotel closer to the conference, and I'll know which corners to avoid organizing dinner on. But it definitely discouraged me from going out to Albuquerque nightlife after conference activites. It felt much more tense than walking around the neighbourhoods of Portland at Rustconf 2019, admiring all the gardens and checking out various little bars.

Flying from Austin to Albuquerque was really easy, and the ABQ airport was really nice. They even have machines that can scan your entire bag, so you don't have to unpack all your electronic devices in the security line.

# Vibe check

My first Rustconf was 2019, and I haven't been back until this year's. In 2019 I was a pretty new Rust programmer, and although I was there with my Cloudflare coworkers, the conference felt pretty overwhelming and intimidating!

Four years later, I feel much more comfortable talking about Rust with strangers. I know a lot more of the language, I'm not overwhelmed when conversations get really deep and technical. I know what I don't know, so I feel comfortable asking "oh, I've got no idea how Cargo actually works -- why wouldn't that idea be possible?" So I definitely felt more confident going in. I felt confident that I could organize group events with strangers, because I remembered feeling pretty lost at my first conf, and wanting to make sure other new Rustaceans always had a breakfast to go to, even if they didn't know anyone. So personally, I had a great time at this Rustconf. I felt confident and capable of helping others have a good time.

On the other hand, Rustconf definitely felt sadder and downbeat than my previous visit. Rustconf 2019 felt _jubilant_. The opening keynote celebrated all the exciting things that had happened over the last year -- and there was a lot to celebrate. Non-lexical lifetimes had just shipped, which removed a _ton_ of confusing borrow checker edge cases. Async/await was just a few short months away from being stabilized, unleashing a lot of high-performance, massively-scalable software. [Eliza Weisman](http://elizas.website/) was presenting a new async tracing library which soon took over the Rust ecosystem. [Lin Clark](https://www.youtube.com/watch?v=KFpU30xluxo) presented about how you could actually compile Rust into this niche thing called WebAssembly and get Rust to run on the frontend -- awesome! It felt like Rust had a clear vision and was rapidly achieving its goals. I was super excited to be part of this revolution in software engienering.

By contrast, I felt like this year's conference was _defensive_ and maybe somewhat _depressed_. I wanted to give the spirit of Rust a hug and tell it, "hey, I know you've had a tough year". In 2023 alone, there was the

 - Rust trademark fiasco
 - Rustconf closing keynote fiasco
 - Serde shipping precompiled binaries fiasco

which resulted in multiple people leaving various Rust teams and projects. There might be even more stuff that just wasn't as publicly visible! Things got _mean_ and heated this year in a way that to me, an outsider to the Rust project, seemed much more public and messy than previous years. I don't know anyone involved in organizing Rustconf, so this is just my personal impression. But the nonstop messiness of the last year definitely played a part in why the conference felt somewhat _withdrawn_ and slightly _depressed_. It's appropriate and understandable after the year the organizers have had. It's hard for Rustconf to have a jubiliant vibe when the organizers have been alternating writing long apologies, or receiving torrents of "feedback" (some deserved, some unfair, some abusive). The official talks from Foundation or Project members seemed like they had to "play it safe" due to the tension caused over this year. The Rust Foundation should issue a grant to Jo Freeman given how many times [The Tyranny of Structurelessness](https://en.wikipedia.org/wiki/The_Tyranny_of_Structurelessness) was mentioned by Rustconf staff!

I miss the exuberance of Rustconf 2019. I understand why this year felt so different. But I hope that next year's Rustconf has a happier, more functional, less divided Rust community.

On the other hand, the weariness and malaise was limited to people who work on or work for Rust. People like me who just _use_ Rust seemed more excited and happier. I loved the optimism of talks from [Carter Schultz](https://github.com/Carter12s), who was almost jumping in the air during his talk, when he explained how easy it was to write his factory's embedded robotics code in Rust. "It just worked! Somehow Rust lived up to the hype! I finished an insanely ambitious project in a new language and it just worked, no problems!" _That_ was the energy I was hoping to find at the conference. I got the same positive energy from [Tim McNamara](timclicks.dev) and [Barton Massey](https://www.pdx.edu/profile/bart-massey) who are bringing Rust to new users with their educational material. They clearly believe in Rust's future and spent a lot of time at Unconference talking about how to spread Rust's benefits to the rest of the tech world.

It's a tough time for the Rust project and foundation, but there's never been a better time to be a Rust user. Rust lets me make software I simply couldn't have made without Rust -- I'm wrapping C++ projects in safe Rust APIs, I'm automatically generating OpenAPI schemas from my HTTP servers, then automatically generating API clients from the schemas. I'm spending less time than ever on unit tests, because libraries like Serde and Diesel are generating all the tricky code for me. Rust has made me _so_ much more productive, _so_ much more less stressed, and helped me work on products I simply didn't have the training to do safely/securely beforehand. Rust is letting me build software that I believe in, so much faster and without exhausting myself every day from double-checking every little potential problem by hand. The Rust mission -- let you write software that's fast _and_ correct, _productively_ -- has never been more alive. So next Rustconf, I plan to celebrate:

 - All the buffer overflows I didn't create, thanks to Rust
 - All the unit tests I didn't have to write, thanks to its type system
 - All the null checks I didn't have to write thanks to `Option` and `Result`
 - All the JS I didn't have to write thanks to WebAssembly
 - All the impossible states I didn't have to `assert "This can never actually happen"`
 - All the JSON field keys I didn't have to manually type in thanks to Serde
 - All the missing SQL column bugs I caught at compiletime thanks to Diesel
 - All the race conditions I never had to worry about thanks to the borrow checker
 - All the connections I can accept concurrently thanks to Tokio
 - All the formatting comments I didn't have to leave on PRs thanks to Rustfmt
 - All the performance footguns I didn't create thanks to Clippy

And I hope you join me.

[winnow]: https://docs.rs/winnow
[clap]: https://docs.rs/clap