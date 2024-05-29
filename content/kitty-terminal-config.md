+++
title = "My Kitty terminal config"
date = 2024-05-29
description = "I made Kitty pretty"

[taxonomies]
tags = ["video", "cli", "dev-ex"]
+++

I've been using [Kitty] terminal for five years now, and I'm really happy with it. Recently I got curious about how to make it look prettier (inspired by all the beautiful terminals I see in some programmer subreddits). So, keep reading for a little explanation of my Kitty config file.

I also made a video showing how I configure a totally fresh Kitty terminal from nothing.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Bgr-vVLYAyc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

But if you prefer text, keep reading.

<!-- more -->

Firstly, I like using programmer ligatures, so that `fn my_func() -> String` the arrow actually uses an arrow glyph. This feature is what originally pulled me towards Kitty terminal in the first place (iTerm 2 didn't support both GPU rendering and ligatures simultaneously, but Kitty did).
Fira Code is a nice font my friend and mentor Areg showed me when I started working, so let's use it.

```
font_family     FiraCode Nerd Font Mono
font_size       14.0
```

Next, let's make Kitty look pretty. I use its multiplexed windows very often. By default they look kinda ugly, but I really like the way they look now.

```
window_margin_width     10
background_image        /Users/adamchalmers/Downloads/night_desert.png
background_image_layout scaled
background_tint         0.97
background_tint_gaps    -10.0

window_border_width     1pt
active_border_color     #44ffff
```

I think this looks pretty nice, personally.


![Kitty with the window config described above](/kitty-terminal-config/pretty_kitty.png)


If I only have one multiplexer window, then I don't really need the nice gaps and borders though.
```
single_window_margin_width 0
```
  
Setting the `enabled_layouts` lets me choose the default layout. Personally I prefer the Tall layout. You might not. Check the docs for more options, or press ctrl+shift+l to cycle through the different window layouts and choose your favourite.
```
enabled_layouts Tall, *
```

This makes the Alt key work properly for keyboard shortcuts, which is really helpful for some CLIs like Helix.
```
macos_option_as_alt yes
```

Finally, some custom keybindings. Now f1 opens a new Kitty multiplexer window, just like ctrl+shift+enter would, but in the same directory.
And f2 opens Helix (my preferred editor, currently) in the same window.
```
map f1 new_window_with_cwd
map f2 launch --cwd=current $EDITOR .
```

Finally, I think kitty's tab bar looks ugly by default, but it's easy to make it look nicer.
```
tab_bar_style       powerline
tab_powerline_style slanted
```

This is really just scratching the surface. Kitty has a lot of other config options, you can read its
[docs][kitty-config] for more. I also really like its [SSH support] and inline image display. Let me know
if there's a great feature I should be using that I'm not. You can view my full kitty config and other dotfiles
on [GitHub][dotfiles].

[dotfiles]: https://github.com/adamchalmers/dotfiles
[SSH support]: https://sw.kovidgoyal.net/kitty/kittens/ssh/
[Kitty]: https://sw.kovidgoyal.net/kitty/
[kitty-config]: https://sw.kovidgoyal.net/kitty/overview/#configuring-kitty
