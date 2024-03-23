+++
title = "Video: My VM and its time-travelling debugger"
date = 2024-03-22
description = "How and why I'm building a compiler"

[taxonomies]
tags = ["rust", "programming", "video", "kcl", "compilers"]
+++

My work at [Zoo] has recently focused on [KCL], our CAD programming language. Currently our [modeling app] uses a tree-walk interpreter, but for various reasons we've been exploring a proper compiler instead. I've been developing the compiler, called [Grackle](https://github.com/KittyCAD/modeling-app/tree/main/src/wasm-lib/grackle). It compiles to a bytecode VM (i.e. an abstract machine), called the KCVM. I gave a 10-minute presentation about KCVM and its time-travelling debugger at a tech talk in Austin recently. Here's the recording!

Thanks very much to [Jam.dev](https://jam.dev/) for hosting the event!

<iframe width="560" height="315" src="https://www.youtube.com/embed/iVeKGU29aSI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

<!-- more -->

[Zoo]: https://zoo.dev
[Winnow]: https://docs.rs/winnow
[KCL]: /tags/kcl/
[modeling app]: https://zoo.dev/modeling-app
