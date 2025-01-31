---
title: 'Introduction to the Jujutsu VCS'
author: Kuba Martin
type: posts
date: 2025-01-31T00:00:01+01:00
images: ["/images/jj/preview.jpg"]
tags:
  - vcs
  - jj
---

[Jujutsu](https://github.com/jj-vcs/jj) (jj), a new version control system written in Rust, has popped up on my radar a few times over the past year. Looked interesting based on a cursory look, but being actually pretty satisfied with Git, and not having major problems with it, I haven't checked it out.

That is, until last week, when I finally decided to give it a go! I dived into a couple blog posts for a few of hours, and surprisingly (noting that we're talking about a VCS) I found myself enjoying it a lot, seeing the consistent design, and overall simplicity it managed to achieve. This post is meant to give you a feel for what's special about jj, and also describe a few patterns that have been working well for me, and which really are the reason I'm enjoying it.

Before we dive in, one last thing you should take note of, is that most people use jj with its Git backend. You can use jj with your existing Git repos and reap its benefits in a way that is completely transparent to others you're collaborating with. Effectively, you can treat it like a Git frontend.

## Undo

Before I get to the meat, something that will surely be very useful for any of your experimentation with jj, and something I would've loved to have had in Git when I was learning it - you can undo any jj operation using `jj undo`, and view an operation log using `jj operation log`.

## Changes

Changes are the core primitive you'll be working with in jj. In Git you had commits, in jj you have changes. A jj change is, however, *mutable*. A change can be modified. It's ID, however, is immutable and randomly generated - it stays constant while you iterate on the change. A change refers to a *revision* (otherwise called a snapshot) which for our purposes is always a Git commit. As you keep working on and modifying a change, the commit SHA it refers to will change.

So to recap - immutable change IDs, mutable changes, mutable revision IDs (Git SHAs), immutable underlying revisions (snapshots - commits). Changes may eventually be marked as immutable - by default this happens (more or less) when they become part of the main branch or a Git tag.

While in Git you generally organize your commits in branches, and a commit that's not part of a branch is scarily called a "detached HEAD", in jj it's completely normal to work on changes that are not on branches. `jj log` is the main command to view the history and tree of changes, and will default to showing you a very reasonable set of changes that should be relevant to you right now - that is (more or less) any local mutable changes, as well as some additional changes for context (like the tip of your main branch).

Changes in jj *can* be marked with bookmarks (what jj calls branches), but you'd generally do that only for the purpose of pushing to a remote Git server, like GitHub, not local organization.

Let's see a sample of a `jj log` invocation:

{{< rawhtml >}}
<pre class="terminal-snippet">
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:green;">@</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:purple;">w</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">twtpovp</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;color:olive;">me@kubamartin.com</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">2025-01-29 23:46:57</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:blue;">6</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">dae3649</span><span style="font-weight:bold;"></span>
│  <span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:green;">(empty)</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:green;">(no description set)</span><span style="font-weight:bold;"></span>
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">◆</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">n</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">zvlmkly</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-29 23:33:23</span> <span style="color:purple;">main</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">e</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">04f25b9</span>
│  add '0000'
│ ○    <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">u</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">knznrnn</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-29 23:43:20</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">2</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">70fa7f9</span>
│ ├─╮  <span style="color:green;">(empty)</span> a merge
│ │ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">v</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">sosyttm</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-29 23:37:52</span> <span style="color:purple;">branch-2</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">35</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">8031f0</span>
│ │ │  add '1111'
│ ○ │  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">r</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">plvonvm</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-29 23:37:10</span> <span style="color:purple;">branch-1</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">12</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">608854</span>
│ │ │  add 'dddd'
│ ○ │  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">m</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">oquvotw</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-29 23:36:43</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">b</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">49e6156</span>
├─╯ │  add 'cccc'
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">◆</span>   │  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">p</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">lnpvmqs</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-29 23:30:48</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">3f</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">373f8e</span>
├───╯  add 'bbbb'
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">◆</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">l</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">ummnokp</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-29 23:30:32</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">10</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">604a80</span>
│  add 'aaaa'
~
</pre>
{{< /rawhtml >}}

{{< rawhtml >}}
<div style="font-size: 0.9em; color: #666; margin-top: -10px; margin-bottom: 15px; font-style: italic;">
Quick tip: you can use <a href="https://github.com/theZiz/aha" style="color: #0366d6;">aha</a> to convert colorized shell output to HTML.
</div>
{{< /rawhtml >}}

The IDs on the left side are change IDs, while on the right size you have revisions (commit SHAs). Unique prefixes of those IDs are marked in color, and you can use those prefixes instead of the full form when referring to changes. A change can be reffered to in jj commands by its ID, the underlying commit SHA, or any bookmark referring to it.

There is one more, slightly crazy, thing about jj changes. Git has a special feature called the Git index to hold any as of yet uncommitted changes (where you `git add`, and then `git commit` them). In jj any files you modify are always in the scope of a change. Any modifications you make automatically become part of the current change, which is called the working copy. In the `jj log` output, this working copy is indicated by the `@` symbol. You create a new working change using `jj new` (by default as a child of the current change), and give it a description using `jj describe` (it starts out having an empty message). You can use `jj status` to get the metadata of the current change.

This much said, here's a quick demo of changing a file as part of a new change.

{{< rawhtml >}}
<script src="https://asciinema.org/a/XpE2I2f0bYp7EYudt1hH9lWlN.js" id="asciicast-XpE2I2f0bYp7EYudt1hH9lWlN" async="true"></script>
<div class="asciinema-caption">Editing a file in a jj repo</div>
{{< /rawhtml >}}

`jj` on its own is an alias for `jj log`.

{{< rawhtml >}}
<div class="info-box">
    <div class="info-box-icon">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="10"></circle>
            <line x1="12" y1="16" x2="12" y2="12"></line>
            <line x1="12" y1="8" x2="12.01" y2="8"></line>
        </svg>
    </div>
    <div class="info-box-content">
        Most jj commands default to operating on the current working change, but let you operate on an arbitrary change (or set of changes!) via the <code>--revision / -r</code> flag.
    </div>
</div>
{{< /rawhtml >}}

### Pattern: git stash

This whole thing about the working copy being a Change may sound weird, but it brings with it an important feature - you can operate on the working copy like on any other change. Personally, I'm a frequent user of `git stash`. When I'm working on something, I often want to pause for a moment and work on / try something else, only to later come back to what I was working on (while leaving that other experiment as yet another stash).

In jj, I can just `jj new @-`, which will start a new change from the parent of my current working copy - leaving my working copy as a normal change with all its file modifications "committed", so I can later come back to it via `jj edit`.

{{< rawhtml >}}
<div class="info-box">
    <div class="info-box-icon">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="10"></circle>
            <line x1="12" y1="16" x2="12" y2="12"></line>
            <line x1="12" y1="8" x2="12.01" y2="8"></line>
        </svg>
    </div>
    <div class="info-box-content">
        The <code>-</code> after the <code>@</code> means "go one level up". It's basically "take the working copy change, and go to its parent". We'll come back to this briefly later, but suffice it to say that there's a whole sensible expression language here, and <code>@--</code> is similarly valid to refer to your grandparent change.
    </div>
</div>
{{< /rawhtml >}}

## Editing Changes in Weird Places

Another cool thing about changes is that you can freely edit any mutable change. `jj edit <id>` lets you "check out" and edit a given change. You can also squish a new change right after another (between it and its children). You can then keep modifying files in the context of that change, and any descendants will automatically be rebased on top of it (with their underlying commit SHAs likely changing, but change IDs staying the same). That sounds a bit scary if you're used to Git (and dealing with conflicts in the middle of a rebase) but fear not, it's actually pretty seamless, and we'll come back to it later.

You can pass `-A` and `-B` to `jj new` to indicate that you want to squish a new change after, or before, another change.

With this, when I notice a mistake in a change 3 levels back, I can just `jj edit <that-change-id>` (with my working copy remaining there for me to come back to), make a fix, which will auto-rebase all following changes (including my original working change), and then `jj edit` to go back to my original working change.

{{<rawhtml>}}
<script src="https://asciinema.org/a/sGxxrDzQJPVbsgraq1HiYbj1j.js" id="asciicast-sGxxrDzQJPVbsgraq1HiYbj1j" async="true"></script>
<div class="asciinema-caption">Editing previous changes with automatic rebase</div>
{{</rawhtml>}}

Occasionally you also need to rebase (move) a set of changes onto some change x, you can do that by using `jj rebase -s <change-id-to-rebase> -d <destination-change>`. The `-s` will bring all descendants along with the rebased change, and there are other variations of this command for different scenarios. E.g. `jj rebase -b <branch> -d <destination-change>` will rebase the entire given branch onto a change, and with no arguments it just defaults to `-b @`, so the current branch. In other words, to rebase the current branch onto main, it's enough to run `jj rebase -d main`.

### Pattern: Squishing a Fix Before the Current Change

You noticed a mistake in your last change, don’t want to fix it as part of this one, and instead want to squish a fix right before it, but you have already done some "uncommitted" work? Just do `jj new -B @`. The `-B` means it’s squished before the current working change (referred to via `@`), and after the previous one (its parent). After you make the fix, you can go back to the original working change via `jj edit @+` (`+` is the opposite of `-`). You could also make the fix in the original working change and run `jj split --interactive` to use your favorite diff ui to select what to push down to a separate change placed right before the current one.

## Bookmarks

In jj, instead of branches we have bookmarks. You create them using `jj bookmark create <name>` (abbreviated to `jj b c <name>`). You update a bookmark to point the current change using `jj bookmark set <name>`. You track remote bookmarks (which will create local corresponding bookmarks that update on `jj git fetch`) by doing `jj bookmark track <name>`.

When you add changes on top of a change that a bookmark is attached to, the bookmark won't automatically move to your new change (like it would with a branch in Git), you have to `jj bookmark set` it manually. You can push bookmarks attached to the current change via `jj git push`. `jj git fetch` will fetch updates to bookmarks from your remote (so generally updates to branches).

### Pattern: Stacked PRs

A common use case with services like GitHub is to split up a big change into multiple PRs, let's say Multipart 1, Multipart 2 and Multipart 3 (from branches multipart-1, multipart-2, and multipart-3 respectively). Each of them is based on the previous one, so you effectively have the following graph, with bookmark pointers on the way:

{{<rawhtml>}}
<pre class="terminal-snippet">
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:green;">@</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:purple;">vq</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">noywln</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;color:olive;">me@kubamartin.com</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">2025-01-30 18:57:23</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:blue;">50</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">e48038</span><span style="font-weight:bold;"></span>
│  <span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:green;">(empty)</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:green;">(no description set)</span><span style="font-weight:bold;"></span>
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">◆</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">sz</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">myonyt</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 18:47:38</span> <span style="color:purple;">main</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">a7</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">9e03bf</span>
│  another change on main
│ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">y</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">otmyvom</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 18:57:16</span> <span style="color:purple;">multipart-3</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">58</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">11d6f6</span>
│ │  add 'hhhh'
│ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">z</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">zxlmyqq</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 18:57:16</span> <span style="color:purple;">multipart-2</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">4</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">ca0376a</span>
│ │  add 'gggg'
│ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">sk</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">noxoxv</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 18:57:16</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">7</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">9356f49</span>
│ │  add 'eeee' and 'ffff'
│ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">vt</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">rtywkl</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 18:51:08</span> <span style="color:purple;">multipart-1</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">5e</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">dec115</span>
│ │  add 'cccc' and 'dddd' (broken)
│ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">ss</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">vuqlmr</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 18:27:29</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">a5</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">9123a7</span>
├─╯  add 'aaaa' and 'bbbb'
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">◆</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">n</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">nlnvpml</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 18:27:03</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">d</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">ee7ba99</span>
│  initial
~
</pre>
{{</rawhtml>}}

Now let's say Multipart 1 got reviewed and you need to fix something. In Git, once you add a commit to it (or modify an existing one), you would have to manually rebase all the other branches. Annoying!

How does jj help us here? We can just run `jj edit vt` (`vt` is a unique prefix of the broken change id), or `jj new -A vt`, make the fix, and run `jj git push -b "glob:multipart-*"`. Everything will be automatically rebased, and all the bookmarks will be updated and pushed. Effortless!

{{<rawhtml>}}
<script src="https://asciinema.org/a/UcUKhLlIyiAaE6iRXu4OdL1E7.js" id="asciicast-UcUKhLlIyiAaE6iRXu4OdL1E7" async="true"></script>
<div class="asciinema-caption">Making fixes in stacked PRs</div>
{{</rawhtml>}}

## Merges

Merges in jj are pretty boring - in a good way. You just create a change with multiple parents. E.g. `jj new x y z` will create a change with three parents.

## Conflicts

One topic I haven't mentioned yet, and is surely by now giving you an unsettling feeling deep inside about all I've said before, are conflicts. With all this rebasing, that's **got** to become a pain, right? Well, it doesn't!

As opposed to Git, where conflicts kind of break your workflow, in the sense that you have to resolve them prior to doing anything else, jj handles conflicts in a first-class manner. A change can just "be conflicted". You can switch away from a conflicted change, you can create a new change on top of an existing conflicted change (and that will in turn also start out conflicted), you can edit a conflicted change, you can do anything. `jj status` and `jj log` will mention the conflict, but it won't block you.

{{<rawhtml>}}
<pre class="terminal-snippet">
<span style="color:green;">~&gt;</span> <span style="color:#005fd7;">jj</span><span style="color:dimgray;"></span> <span style="color:#00afff;">log</span>
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:red;">@</span>    <span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:purple;">u</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">uxrrmuw</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;color:olive;">me@kubamartin.com</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">2025-01-30 19:54:21</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:blue;">d</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">7f2700a</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:red;">conflict</span><span style="font-weight:bold;"></span>
├─╮  <span style="font-weight:bold;">merge</span>
│ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">q</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">lrwtlny</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 19:44:09</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">c</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">ae28ccf</span>
│ │  add 'cccc'
○ │  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">r</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">qtrswmw</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 19:43:59</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">5</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">6d6ab4a</span>
├─╯  add 'bbbb'
○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">k</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">nklsqnp</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 19:43:59</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">1</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">0b01706</span>
│  add 'aaaa'
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">◆</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">z</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">zzzzzzz</span> <span style="color:green;">root()</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">0</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">0000000</span>
<span style="color:green;">~&gt;</span> <span style="color:#005fd7;">jj</span><span style="color:dimgray;"></span> <span style="color:#00afff;">status</span>
Working copy changes:
<span style="color:green;">A typescript</span>
<span style="color:red;">There are unresolved conflicts at these paths:</span>
file.txt    <span style="color:olive;">2-sided conflict</span>
Working copy : <span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:purple;">u</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">uxrrmuw</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:blue;">d</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">7f2700a</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:red;">(conflict)</span><span style="font-weight:bold;"> merge</span>
Parent commit: <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">r</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">qtrswmw</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">5</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">6d6ab4a</span> add 'bbbb'
Parent commit: <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">q</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">lrwtlny</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">c</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">ae28ccf</span> add 'cccc'
</pre>
{{</rawhtml>}}

The conflict will be represented in the code via conflict markers:

{{<rawhtml>}}
<pre class="terminal-snippet">
aaaa
<<<<<<< Conflict 1 of 1
%%%%%%% Changes from base to side #1
+bbbb
+++++++ Contents of side #2
cccc
>>>>>>> Conflict 1 of 1 ends
</pre>
{{</rawhtml>}}

and you can either manually resolve this conflict by editing the code itself, or use `jj resolve` to bring up your favorite three-way-merge tool (e.g. I've configured my jj to bring up a visual conflict resolver in Goland) and resolve it there.

Once you fix the conflict in a change, all descendants of this change will also cease being conflicted. You could leave a change conflicted and only resolve the conflict in a follow-up child change - that is a completely valid and supported approach.

{{<rawhtml>}}
<script src="https://asciinema.org/a/xwJxb5ue1pRJt6Iw1JZZhAHyN.js" id="asciicast-xwJxb5ue1pRJt6Iw1JZZhAHyN" async="true"></script>
<div class="asciinema-caption">Merge conflict resolution</div>
{{</rawhtml>}}

### Pattern: Working on Two Things at the Same Time

I've mentioned stacked PRs, but another situation you might have is working on two independent things in parallel. Let's say on bookmarks `thing-1` and `thing-2`.

In order to have both things simultaneosuly active in your codebase, you can create a development change via `jj new thing-1 thing-2 -m "dev"`, that will be a merge between both of them, but will stay local and you'll never push it (`-m` just lets you give a description to a change when creating it, without a separate `jj describe` invocation). You will however `jj edit` this development change to do your work.

{{<rawhtml>}}
<pre class="terminal-snippet">
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:green;">@</span>    <span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:purple;">zv</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">tqnnxv</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;color:olive;">me@kubamartin.com</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">2025-01-30 22:42:29</span><span style="font-weight:bold;"> </span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:blue;">5</span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:dimgray;">28d6811</span><span style="font-weight:bold;"></span>
├─╮  <span style="font-weight:bold;">dev</span>
│ ○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">zm</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">smpynx</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 22:36:26</span> <span style="color:purple;">thing-2</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">2</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">55cea26</span>
│ │  changes to file2
○ │  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">v</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">nxwrykm</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 22:35:43</span> <span style="color:purple;">thing-1</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">d</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">bcec585</span>
├─╯  changes to file1
○  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">p</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">uxnvzsm</span> <span style="color:olive;">me@kubamartin.com</span> <span style="color:teal;">2025-01-30 22:34:43</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">0b</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">98fe5b</span>
│  initial
<span style="font-weight:bold;"></span><span style="font-weight:bold;filter: contrast(70%) brightness(190%);color:teal;">◆</span>  <span style="font-weight:bold;"></span><span style="font-weight:bold;color:purple;">zz</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">zzzzzz</span> <span style="color:green;">root()</span> <span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">00</span><span style="filter: contrast(70%) brightness(190%);color:dimgray;">000000</span>
</pre>
{{</rawhtml>}}

Then, whenever you want to move some modifications into one of the branches, you can use `jj squash --into <target-change-id> <files>` to move all modifications to a set of files down into one of the branches. There's also `--interactive` where you can use a diff tool to choose modifications to squash into another change, and finally there's a [newer `jj absorb` command](https://jj-vcs.github.io/jj/latest/cli-reference/#jj-absorb) which can automate this process in certain scenarios.

{{<rawhtml>}}
<script src="https://asciinema.org/a/VAW2jZSzY3Z5qKYvhvBNLfjEf.js" id="asciicast-VAW2jZSzY3Z5qKYvhvBNLfjEf" async="true"></script>
<div class="asciinema-caption">Editing files in a merged dev-change and<br> selectively squashing the changes into branches</div>
{{</rawhtml>}}

In conclusion, you can keep working in a local-only merge-change of your branches, and selectively push down any modifications to the relevant branch (This setup would've seemed pretty scary before jj, right? I hope it's a bit less scary now.), and then push just those branches themselves to the remote.

## Revset Expressions

jj commands operate on revisions or sets of revisions (revsets). You can refer to those directly, or use a special expression language to describe them. You've seen me refer to a change previously via `@-`. That was a very simple expression that evaluated to the parent.

There is, however, much more. There are functions - like `parents(x)` to get the parents of a change - and operators - like `x+` to refer to the child of x, or `x::` for all descendants of x including x.

`jj log` accepts a revset expression, so you can use it to experiment with them. The default revset it displays is also configurable. Overall, the expression language is powerful and consistent, with simple things being generally easy, and harder things being (presumably, I've honestly spent too little time with it) possible.

See [this article](https://v5.chriskrycho.com/essays/jj-init/#revisions-and-revsets) for a much more extensive exploration of the jj expression language, and the [jj docs](https://jj-vcs.github.io/jj/latest/revsets/) themselves.

### Pattern: Partial Stashes

Another kind of stash I occasionally like to do is partial, where I temporarily roll back changes to a set of files to verify the before-after (e.g. confirm that a passing test was failing before).

In jj the split command works well for this. Just `jj split --parallel modifiedFile.txt` will move the file into a parallel change. You can do whatever you want to do, and later run `jj squash --from parallel_change_id` to get the file modifications back into the current change.

{{<rawhtml>}}
<script src="https://asciinema.org/a/SYe9MBwShu7uWuZJb2vJDffrk.js" id="asciicast-SYe9MBwShu7uWuZJb2vJDffrk" async="true"></script>
<div class="asciinema-caption">Partial stash using <code>jj split --parallel</code></div>
{{</rawhtml>}}

## Setting Up jj with an Existing Git Repo

It's trivial to start using jj with an existing git repo, though I'd advise cloning it fresh into a new directory.

In a directory where you already have a git repo, you can just run `jj git init --colocate`. Your `.git` directory will stay in place, and jj will keep it updated, so e.g. your editor won't be confused what's happening. It integrates fairly well, with e.g. the working change - even though it's backed by a commit - being presented as the git index, so your editor can still show files "modified in this change".

## Should You Switch?
The cost of switching is low, as it integrates seamlessly with your existing workflow. It frankly also takes a day tops to get used to, and there’s something to be said for using *nice* things. Sure, you can make excellent tea in any food-safe kettle, but if you have a nice tea kettle, you’ll enjoy it every time you make tea. I use my vcs quite a lot, so why not make that pleasant too?

## Conclusion

I hope the above gave you an intuition for what Jujutsu, the VCS, is all about, and ideally even encouraged you to take a look at it.

If you'd like to do some more readings about jj, I've used the below articles and guides when learning it, and a lot of what I wrote above is inspired by parts of them. Check them out!

- https://v5.chriskrycho.com/essays/jj-init
- https://neugierig.org/software/blog/2024/12/jujutsu.html
- https://tonyfinn.com/blog/jj/
- https://v5.chriskrycho.com/journal/jujutsu-megamerges-and-jj-absorb/

Finally, I of course recommend just reading the docs: https://jj-vcs.github.io/jj/latest/
