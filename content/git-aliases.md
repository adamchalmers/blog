+++
title = "Missed opportunities for git aliases"
date = 2023-06-19
description = "Analyse your shell history, to see where aliases could save you time"

[taxonomies]
tags = ["cli", "tools", "dev-ex"]
+++

I don't pair program very often, but sometimes when you're stuck on a really annoying problem, it can be helpful to have a coworker looking over your shoulder. My new job is fully-remote, so we occasionally screenshare when someone's stuck. One of my favourite things about pair programming is that I get to see the little idiosyncracies of everyone's workflow. For example, you notice which peers use vim for everything, and pick up on all their awesome little keybindings to jump around the document. Or you realize that one peer has installed a bunch of awesome VSCode extensions that you can steal.

Several coworkers have remarked that I use shell aliases way more often than them! For example, I mostly work with rust, so I used `alias c="cargo"` to tell my shell "when I type `c`, replace it with `cargo`". Then I use Cargo's built-in aliases, e.g. `cargo check` and `cargo c` are equivalent. So, when I want to run `cargo check` I just type `c c` and my shell expands it to `cargo c` and Cargo expands it to `cargo check`. My coworkers were very amused by this! I think a suite of nice, easy aliases can really save you time and avoid interrupting your [flow state][flow]. I'm trying to use more aliases for common tasks, and here's how I've been doing it.

<!-- more -->

# Git aliases in oh-my-zsh

Besides cargo, the main CLI I tool I use is git. Ah, git. Some love it, some hate it, [some both][jyn-git]. I spend so much time in git across every single project and job that I've come to rely heavily on git aliases. I use zsh as my shell, and almost all zsh users use [oh-my-zsh] as well. oh-my-zsh is a plugin ecosystem for zsh. I don't go too crazy, but I do really like their git plugins, which let you show your git status in your prompt.

![My shell prompt, showing current directory, git status and branch](/git-aliases/prompt.png)

oh-my-zsh also comes with a list shell aliases for common git commands. E.g. they set `gaa` to be `git add --all`. You can see the full list [here][zsh-git]. It's worth taking a quick look through them to learn the common ones, they really save you time. In fact, I can prove it! Let's look at the 20 most common shell commands I type.

```sh
history | rg "[0-9]+ +(.*)" -or '$1' | sort | uniq -c | sort -r | head -n 20
```

This shell command opens my shell history, parses each line with a regular expression and outputs only the command part (i.e. excludes the line number), then counts how many times each command appeared, sorts it from most-common to least-common, and then shows the top 20. Here are the results:

```
1049 gs
 440 gd
 434 gaa
 314 c c
 312 g pull
 278 gco main
 278 ..
 251 g log
 245 g push
 208 gc --amend
 205 gpf
 137 g restore .
 111 g submodule update --init --recursive
 109 gco -
  85 ls
  80 exit
  75 code .
  62 c t
  61 c c --tests
  56 make kittycad
```

Yep, the three most common commands I type are `gs`, `gd` and `gaa`, which are oh-my-zsh aliases for `git status`, `git diff` and `git add --all` respectively. Then there's `c c` (cargo check) and more git aliases. 

# Adding your own aliases

If you want to add your own aliases, you have two options:

1. Add a shell alias, like oh-my-zsh does
2. Use git's built-in alias system

Yeah, git has its own built-in aliasing system! Here's the `alias` section of my `~/.gitconfig` file.

```
[alias]
    st = status
    sw = switch
    c  = commit
    rb = rebase
    pl = pull --recurse-submodules
    pf = push --force-with-lease
```

(note the `--force-with-lease`, if you haven't seen that, I strongly suggest [this quick read about it][gpf])

So, if I type `git st`, git itself will expand that to `git status`. Actually I only type `g st` because oh-my-zsh aliases `g` to `git` by default. _Actually_ I use `gs` because oh-my-zsh has its own alias for that. But still, it's good to know that git has its own alias system, in case you aren't using oh-my-zsh, or you're logged into a remote machine, or using a coworker's computer, or whatever.

# Finding chances for new aliases

Recently I wondered if I was using enough aliases. In this economy, you have to be relying on a series of arcane shell configurations to maximize your productivity and impress your coworkers. If you save a few nanoseconds every day, over the lifespan of the universe, that will add up to literally hours of saved time! So let's check if I can use more aliases. 

```sh
history | rg "[0-9]+ +(g.*)" -or '$1' | sort | uniq -c | sort -r | head -n 20
```

This command is very similar to the one above. But I'm checking the history for any command that starts with `g`. There might be some false positives in there, e.g. the `go` command might get included. But I would rather ignore those lines than try to write a really complicated regex, so let's just pretend it's all fine. Here's the results:


```
1049 gs
 440 gd
 434 gaa
 312 g pull
 278 gco main
 251 g log
 245 g push
 208 gc --amend
 205 gpf
 137 g restore .
 111 g submodule update --init --recursive
 109 gco -
  36 g reset HEAD~
  34 g stash
  32 g show HEAD
  29 g rebase -
  26 gd --cached
  26 g rebase main
  24 g reset --hard
  23 g stash pop
```

A few things jump out:

 - As mentioned above, the zsh aliases are really helpful for common tasks
 - I type `g pull` very often, but surely there's already an alias for that? Yep, oh-my-zsh aliases `gl=git pull`, I should use that more.
 - Same with `g log`, I should use the zsh alias `glg` instead.
 - There are some commands where I'm typing a lot, but zshell either doesn't have aliases, or their aliases set options I don't want. I should build aliases for them: 
  - `gc --amend`
  - `g submodule update --init --recursive`
  - `git restore .`

So, this showed me which lines I type often enough that it's worth creating aliases and training myself to use them. I've added these aliases:

```
ca = commit --amend
f  = commit --amend --no-edit  # pronounced internally as "git fixup"
rd = restore .
pf = push --force-with-lease
su = submodule update --init --recursive
```

# Why bother?

Why bother with all this? Well, I think it's worth analyzing your own shell history, because everyone uses their terminal differently. Your common commands are going to be different to my common commands, and you deserve to have a beautiful, luxury, personalized, fully-automated terminal experience. Building aliases for commands you actually use frequently will be more useful than just copying some list of aliases from someone else's setup.

I think aliases are Good, Actually. You're staring at the terminal all day, if you can't give your eyes a break, you might as well give your fingers a break. You can tweak the above regexes to search across all commands, or only git commands, or whichever other tools you commonly use (e.g. `kubectl`).

I was thinking about aliases because I'm trying to spend some time this year becoming more efficient at the things I do every day. Partly inspired by [Dan Luu's post about velocity](https://danluu.com/productivity-velocity/). E.g. I decided that this is the year I actually get good at vim. Just learning a few basics has really sped up my editing (I use mostly VSCode with the vim extension).

Another big reason is that it's just cool. Having a series of arcane invocations you type into a terminal to execute long chains of logic is just really fun. Does it save time? Probably. Does it save much time? Eh, maybe. But it's undeniably fun to have a tiny little keystroke do a long series of tasks you precisely programmed it to do.

Anyway, if you have other tips for saving time in your terminal or editor, I'd love to hear about them in the comments or on [my twitter/mastodon/bluesky](/about).

[jyn-git]: https://jyn.dev/2022/09/02/git-cheats.html
[flow]: https://en.wikipedia.org/wiki/Flow_(psychology)
[zsh-git]: https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/git/git.plugin.zsh
[oh-my-zsh]: https://ohmyz.sh/
[gpf]: https://mtsknn.fi/blog/how-to-force-push-in-git-with-style-and-some-safety/