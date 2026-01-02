+++
title = "2025 reflections"
date = 2026-02-01
description = "Things I did, read, enjoyed, etc"
draft = false

[taxonomies]
tags = ["personal", "career"]
+++

It's been a good year. We moved back to Austin from New Orleans, I've watched my first kid grow and develop and really start having a personality, and I've had a lot of fun at work.

<!-- more -->

# Books read

Since moving to the USA in 2017, I've logged every book I read on [my Goodreads](https://www.goodreads.com/user/show/19657142-adam). This year, I read


**Fiction**

 - A Fire upon the Deep (Vernor Vinge, reread)
 - The City and its Uncertain Walls (Haruki Murakami)
 - A Deepness in the Sky (Vernor Vinge)
 - Moby Dick (Herman Melville)
 - Shards of Earth (Adrian Tchaikovsky)
 - Inherent Vice (Thomas Pynchon, reread)
 - Shadow Ticket (Thomas Pynchon)
 - We Live Here Now (Sarah Pinborough)
 - The Priory of the Orange Tree (Samantha Shannon)
 - Rose/House (Arkady Martine)

**Nonfiction**

 - A Rome of One's Own (Emma Southon)
 - Autocracy Inc (Anne Applebaum)
 - Not the End of the World (Hannah Ritchie)

**Nonfiction, Chinese history**

 - Empress Dowager Cixi (Jung Chang)
 - China and the Manchus (Herbert A. Giles)
 - Emperor of China: Self-Portrait of Kang-Hsi (Jonathan D. Spence)
 - Tea Horse Road (Michael Freeman, Selena Ahmed)
 - Imperial China 900-1800 (F.W. Mote)
 - Silk, Silver, Opium (Michael Pembroke)
 - The Search for Modern China (Jonathan D. Spence)

Of these, my least favorite was probably Moby Dick. It's got some amazing paragraphs separated by chapters and chapters of boredom. Sorry, I gave it a try, and I persisted to the end, but I mostly used it to fall asleep when I was awake late at night.

I read Empress Dowager Cixi on a whim, because I was at a used book store, and found a copy. I'd heard [Jung Chang on Dwarkesh's podcast](https://www.dwarkesh.com/p/jung-chang), and really enjoyed her interview, so I was keen to read something by her. Wow! What a great book. It really opened up my interest in Chinese history, which clearly I became pretty passionate about this year. I really enjoyed learning about that region of the world, and I hope to read more Chinese history next year. I've been enjoying [Amimisu on YouTube](https://www.youtube.com/@Amimisu) who tells a lot of interesting stories from Chinese history. It's been fun to listen to her while I cook food or wash dishes.

This was my fourth time reading Inherent Vice. Every time I've read it, I've gotten something different from it. This time, I was mostly focused on the idea that idealistic heroes want to change the world, but the system is so powerful that it subverts your energy, making you work against your own goals, and ultimately undoing your work. But despite that, you shouldn't become despondent and give up, or cynically conclude that nothing can ever improve. Because even if you can't save the world, you can probably save a life, and that's not a consolation prize. It's just as meaningful.

# Work done

It was a pretty strong year for [Zoo](https://zoo.dev). We launched 1.0 of Zoo Design Studio, and have actual revenue now. Very exciting. For me, launching 1.0 meant that KCL committed to a no-breaking-change policy. When you open your KCL files in Zoo, even if we've automatically updated the app, your old files should continue to produce the same model as they always have.

This meant we needed to lock down the language. I knew we didn't want to have multiple versions of KCL with different features, so we decided to use keyword arguments. This was a very ambitious project, we basically rewrote the entire KCL standard library, and all the code mods that power the frontend. Extruding a surface no longer looks like `extrude(surface, 4cm)`. Now it looks like `extrude(surface, distance = 4cm)`. This means that as we add new features to the language, they just become extra arguments, so you can do `extrude(surface, distance = 4cm, twistAngle = 40deg)` etc. New features are added as optional arguments, with sensible defaults if they're not provided.

I wrote a complete KCL programming guide as part of our 1.0 release process, the [KCL book](https://kcl-book.zoo.dev/). I was very inspired by my friend Steve Klabnik, who cowrote The Rust Book, and I wanted something similar for KCL. It includes a bunch of interactive 3D visualizations, and there's a corresponding [YouTube video series](https://www.youtube.com/watch?v=pT1BhnyasUA&t). I had never done much video production before, so learning how to film myself, get a decent-looking film setup, and edit my videos was a lot to learn. But it was also really fun.

We also started working on an in-house constraint solver implementation, which I jokingly code-named "ezpz" (because I was very intimidated by how difficult this was going to be). I was really intimidated when starting this project, because I'd never done anything like it before. I wasn't familiar with the math, I wasn't familiar with linear algebra or the complex numeric libraries that people in the field lean on. Over the last few months, with help from really good coworkers like Nick and David, I've finished version 1, and it's performing really well in production.

I loved this constraint solver project because I learned so many new things. I learned about math, about various linear algebra libraries (e.g. [faer](https://docs.rs/faer), about SymPy and how to use it to [differentiate equations and generate Rust code for the derivatives](https://github.com/KittyCAD/ezpz-sympy/commit/f2651f9f69fe40ff5ce657cff95e3032ed122cd3#diff-b10564ab7d2c520cdd0243874879fb0a782862c3c902ab535faabe57d5a505e1R64-R65). How to drive down allocations in the hot path of my code, how to use property-based tests and fuzzers and [cargo mutants](https://mutants.rs). Expect to hear more about ezpz during 2026. 

I did a lot of miscellaneous work around KCL that doesn't fit neatly into a big visible goal. We improved its performance, fixed a lot of small issues in its parser, built a Treesitter grammar for it (thanks to [Oliver](https://github.com/eikopf) for his help with that), added a lot of new features in conjunction with our frontend and engine team, etc. I built a big library for engineering holes with my coworker Ben (you can set countersinks, counterbores, end drill angles, etc etc) in pure KCL, which goes to show KCL is finally suitable for complicated libraries.

The biggest thing we did for KCL is to hire a third person, who'll hopefully be starting in early January. I'm really excited to have a dedicated KCL team of 3 now.

# Conferences

I've now done 4 tech conferences 12 months! Late 2024 I did RustConf and EuroRust, and then in 2025 I did Rust Asia in Hong Kong, then Rust Forge in New Zealand. I've had so much fun travelling the world, meeting smart programmers and learning from them. Thank you so much to the conferences who made this possible. Speaking at a conference has been a goal for a long time, and I'm really thankful for the opportunity.

I loved the conversations I had with people after my talk, at dinner or coffee, or in the hallway between sessions. Thanks so much to Allen Wyma and Tim McNamara for organizing their conferences! I hope I can attend more of them in 2026. It's been especially great to meet friends at these conferences and travel the world together. Meeting up repeatedly at different places in America, Canada, Vienna, New Zealand and Hong Kong is such a fun way to bond with other programmers I admire.

<blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:3ohd5nodgvayjdegkhzayr6q/app.bsky.feed.post/3lxrxarcabk26" data-bluesky-cid="bafyreiao6tjpy5rs5veejhfwtbxwczholcdjvk7chzsfazkl74yy4yzktm" data-bluesky-embed-color-mode="system"><p lang="en">Just hanging out with some cool engineers who inspire me like @ncameron.org @predr.ag and @steveklabnik.com at Rust Forge<br><br><a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q/post/3lxrxarcabk26?ref_src=embed">[image or embed]</a></p>&mdash; Adam Chalmers (<a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q?ref_src=embed">@adamchalmers.com</a>) <a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q/post/3lxrxarcabk26?ref_src=embed">September 1, 2025 at 10:48 AM</a></blockquote><script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script>

<blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:3ohd5nodgvayjdegkhzayr6q/app.bsky.feed.post/3llhc4bbtss2r" data-bluesky-cid="bafyreifbe5mc7ktnjeb3cyzoi3ewpy7577x7jnenomqddq6pevpdccehxi" data-bluesky-embed-color-mode="system"><p lang="en">Got to meet Hai Chi 茌海, creator of the Monoio async runtime<br><br><a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q/post/3llhc4bbtss2r?ref_src=embed">[image or embed]</a></p>&mdash; Adam Chalmers (<a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q?ref_src=embed">@adamchalmers.com</a>) <a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q/post/3llhc4bbtss2r?ref_src=embed">March 28, 2025 at 12:01 PM</a></blockquote><script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script>

# Nonconference travel

We visited Hawaii for a wedding, then Australia for another wedding (and to see my family). Both great trips.

# Austin Systems

When I moved back to Austin, I started [Austin Systems](https://austinsystems.org), a new systems programming meetup. I had so much fun in the New Orleans tech world, which had vibrant local community and weekly meetups. I wanted to bring that warmth back with me. So far, Austin Systems has had 4 events. We run monthly. They've been very successful! Each time, we've found speakers and sponsors (for the pizza and beer). I'm so happy to see a room full of 20-30 people every month, who all want to learn about making new interesting software. I hope Austin Systems grows and has a great 2026. My brother-in-law Turner has been great at helping me coordinate it and host from his office. James and Steve have been great helps getting the pizza each month. Thanks to everyone who's spoken, attended, sponsored or helped.

# Open source

I've never really done much open-source community work. I mean, a lot of my projects have been open source, but run entirely by my employer, not by a community (e.g. a lot of Zoo and Cloudflare's code is open-source, but it's not owned by the community or anything). This year, a friend of a friend started a new big Rust community project called [rv](https://rv.dev). It's the `uv` of Ruby. I had a lot of fun helping the team improve their Rust project. I started by adding tests, benchmarks, fuzzing, nicer CI suites, etc. Then I started helping manage PRs and issues from the community. Now I've been adding new features too! It's really fun to work on a super performance-sensitive community project, where I can polish up my Rust skills on a small project with a very different architecture to my normal paid work. I've also learned a lot about how Ruby packages work, which is going to be really helpful background when we eventually build a KCL package manager.

# My workflow

I spent 2025 mostly programming the same way I did in 2024. [Helix](https://helix-editor.com/) remains my editor of choice. I'm still using git, I haven't (yet) switched to jj (although once [ersc](https://ersc.io) launches, I might have to). Still using zsh. I did install [atuin](https://atuin.sh/) as a nicer shell history tool, but I'm not syncing data or anything. It's just a little more pleasant to use than the built-in shell history.

I spend a lot of time switching between different git branches, so I made a tiny little branch picker that works exactly the way I want it to. It's a little TUI built in Ratatui.

<blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:3ohd5nodgvayjdegkhzayr6q/app.bsky.feed.post/3lscy7ilu3c2q" data-bluesky-cid="bafyreiclzkbb4kawediw4jmqx2odcl6nhvfdh2jsiy5dr6v6qoug3h4ht4" data-bluesky-embed-color-mode="system"><p lang="en">I spend a lot of time running git branch, looking at my branches, figuring out which one I want, then switching to it. So I made a tiny little TUI to combine those steps. You select a branch with arrow keys, then press enter to switch to it, or q to exit with no changes. Made with @ratatui.rs<br><br><a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q/post/3lscy7ilu3c2q?ref_src=embed">[image or embed]</a></p>&mdash; Adam Chalmers (<a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q?ref_src=embed">@adamchalmers.com</a>) <a href="https://bsky.app/profile/did:plc:3ohd5nodgvayjdegkhzayr6q/post/3lscy7ilu3c2q?ref_src=embed">June 23, 2025 at 8:49 PM</a></blockquote><script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script>

# AI

I started this year as a big AI skeptic. I didn't use Copilot or Chatgpt or anything. But over this year, we started using some tool called [Graphite](https://www.graphite.com/) and I was very impressed by its suggestions. It rarely commented anything, but when it did, it was almost always a comment worth reading. It's a very high-signal tool. I even found it helpful for editing the KCL book. It would sometimes point out, "you've said 'let's build a triangle', but the code you supplied actually built a square". This kind of little mistake is really annoying to catch manually. Having a tool that reviewed my drafts and made occasional suggestions like that was very helpful.

This made me curious about broader AI tools, and a lot of programmers I respect (e.g. many employees of Oxide Computer) were publicly talking about how useful the tools had become. So I started chatting about bugs I was fixing with Chatgpt. It didn't always have anything helpful to say, but it only took me a minute or two to describe the problems or ideas I was thinking about, then read its responses. Sometimes they were helpful. I was particularly impressed when I uploaded a flamegraph SVG to Chatgpt and it pointed out some strategies I could use for reducing allocations in the KCL formatter.

In around October I started trialling Codex, and I was again impressed by it. It really can do some good programming. My CEO Jess likes to use it to automate the annoying boring parts of programming, like doing big refactors that change a property to a method, or adding a new `my_field: Default::default()` to every struct instantiation.

So, I started this year a complete AI non-user, but ended it as a twice-or-thrice-a-day AI user. I've been using it to

- help explain other languages
  - e.g. "how do I tell Typescript this is either "a" or "b" and not just string" 
  - e.g. "how do I convert the python list [1, 2] into a SymPy vector with components [1, 2]"
- Give me review on my PRs
  - e.g. "Review the current HEAD commit. Have I introduced any performance problems?"
  - e.g. "Review HEAD, are there any places that I've duplicated code from elsewhere in the codebase, where I should probably make a helper function instead?"
  - e.g. "Review HEAD, which parts of the code could use more comments or better variable names to communicate my intent more clearly?"
  - e.g. "Review HEAD, are there any potential bugs you see that I should investigate?"
- Give a fuzzy summary of large data files
  - e.g. "find any KCL stdlib functions that don't have a @(feature_tree) annotation in them -- this would be very annoying to do as a regex because you have to think about multi-line annotations, other annotations in the block etc.
  - e.g. "analyze this flamegraph. What major red flags do you see here?" This noticed I was allocating a lot of strings in the KCL formatter, and gave me some ideas for changing it to mostly use one big mutable string made it a lot faster.
 - Act as an advanced spellcheck or very stupid copyeditor for my writing
   - e.g. it noticed several places in the KCL book where I'd say "Let's use this function to model a cube" and it would say "you actually modelled a cylinder", or "you said you would make a filletted cube, but you didn't provide any fillet calls".
 - Do boring codegen
   - e.g. "write a table-driven unit test for function foo, which has a vector of test cases, each with fields expected, input and name" -- then I can check the test coverage easily and see if it missed anything.
   - Sometimes I'll start it off with one test case and the structures/loop for calling it, and ask it to fill in more test cases for typical edge cases like 0, null, negative numbers etc.

I don't think the AI can replace what I can do. It's been a helpful tool so far. I am cautious excited to see what happens in the future. I still think they're terrible for writing, and you'll never see a single word on my blog which is generated by AI.

# Family

My daughter is nearly 2 now! She's definitely been the highlight of my year. Having a tiny little girl scream "Dada!" and run to hug my knees is just... the best feeling in the world. I love her to bits.

Our favorite books to read this year have been Madeline, Dr. Seuss and [A is for Alien](https://www.amazon.com/Alien-Century-Studios-Little-Golden/dp/073644484X). She has her own favorites, and I have my own favorites, but those are our joint favorites. I've realized that a lot of childrens books are crap, so the books with both good art and good writing are very special to me. Madeline is definitely my favorite so far.

My wife finished her PhD, and her New Orleans internship, so we moved back to Austin TX. I'm so glad we're back. I really loved my time in New Orleans, and I made some really close friends there. I definitely want to visit again soon. But being back in Austin is such a relief. Austin's local government works surprisingly well. Or perhaps New Orleans' is surprisingly bad. Or both. Either way, it's nice to have trash that gets picked up on time, and power that doesn't randomly crap out during the day, and fast internet, and water treatment systems that don't fail every 3 months, making you boil all your water for 48 hours.

My girl's been going to daycare, and she's really loving it. She's made friends there, and plays with them, and loves her teachers. She's been happily playing with our dog, and cuddling him. God. I love having a baby! I love being a dad.

Because the baby wakes up early, I've been getting to sleep a lot earlier too, so that I can wake up when she wakes up and needs me. After years of being a night owl, I unfortunately have to say: the morning people have it better. I'm not naturally a morning person, but the benefits of being up early are worth it.

 - You can get your morning walk done before the Texas sun gets too hot
 - When you eat early, all the restaurants have free tables, you aren't fighting over reservations
 - You feel superior about yourself
 - I'm more likely to exercise in the morning, and less likely to exercise if I've already had a long day at my job.

# Personal

I didn't do any writing or comedy or creative work this year, which was a shame, but I didn't really think I would get to. Being a dad and working at Zoo have taken up enough of my time. I did make time to work out, at a really fun barbell-focused strength training gym in New Orleans. I've been working out regularly at a new gym near my house in Austin. I don't like it as much as the strength-focused NOLA gym, but that's OK, it's still regular exercise.

I've been cooking a lot more. I work from home, and my wife works in an office, so I've been making breakfast and dinner for my girls. It's been a lot of fun. I stopped being vegetarian this year, so I've got a lot of new food to experiment with. I've enjoyed making whole roast chicken, big salmon platters, and 10-hour slow-cooked lamb. I still love vegetarian food, but it's nice to have more options.

I kept up my Spanish practice, and reached a 1000 day Duolingo streak. Once I reached 1000 days, I deleted the app. It's time for something new. If you have a favorite Spanish language app (or Mandarin), please let me know. I want to try other alternatives.

# Goals for 2026

I don't really have anything new I want to do, or changes I want to make. I think this year is more about continuing the good habits I built last year.

 - Continue to sleep well and exercise regularly, so that I can show up for my family.
 - Continue to take on big projects outside my comfort zone at work, so I can grow as a programmer
 - Continue to have friends over for dinners or parties, because that lets me see my people while also getting the baby to sleep on time.

I would like to learn how to use a debugger properly, because println debugging is just not cutting it anymore.

If I have spare capacity, I'd love to get back into some kind of creative outlet, whether comedy or writing or something else. But that's a nice-to-have. It's more important to maintain the good, stable habits above, so that I can keep showing up for my family.

# Entertainment

I don't remember a lot of what I watched this year, but here's the ones that stuck in my mind.

TV:

- Pantheon: Amazing. A must-watch for sci-fi fans.
- Andor: Amazing. You don't need to like, or know anything about Star Wars. It's incredible.
- House of the Dragon: very good
- Foundation: sometimes very good, sometimes good
- Pluribus: very good

Movies:

- Bugonia: Amazing.
- One Battle After Another: Very good.

Podcasts:

- I got really into Acquired podcast, which made business interesting to me for the first time in my 34 years on planet Earth.

# The end

Have a great 2026 everybody.


