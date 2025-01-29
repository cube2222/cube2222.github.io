---
title: 'Introduction to the Jujutsu VCS'
author: Kuba Martin
type: posts
date: 2025-01-28T17:19:19+00:00
tags:
  - vcs
  - jj
---

[Jujutsu](https://github.com/jj-vcs/jj) (jj), a new VCS, has popped up on my radar a few times over the past year. Looked interesting based on a cursory look, but being actually pretty satisfied with Git, and not having major problems with it, I haven't checked it out.

That is, until a couple days ago, when I finally decided to give it a go! I dived into a couple blog posts for a few of hours, and surprisingly (noting that we're talking about a VCS) I found myself consistenly smiling, seeing the consistent design, and overall simplicity it managed to achieve. This post is meant to give you a feel for what's special about jj, and also describe a few patterns that have been working well for me, and which really are the reason I'm enjoying it.

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
<pre style="font-size:14px;background-color:rgb(18, 19, 20);border-radius:6px;padding:12px;">
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

(quicktip: you can use [aha](https://github.com/theZiz/aha) to convert colorized shell output to html)

The IDs on the left side are change IDs, while on the right size you have revisions (commit SHAs). Unique prefixes of those IDs are marked in color, and you can use those prefixes instead of the full form when referring to changes. A change can be reffered to in jj commands by its ID, the underlying commit SHA, or any bookmark referring to it.

There is one more, slightly crazy, thing about jj changes. Git has a special feature called the Git index to hold any as of yet uncommitted changes (where you `git add`, and then `git commit` them). In jj any files you modify are always in the scope of a change. Any modifications you make automatically become part of the current change, which is called the working copy. In the `jj log` output, this working copy is indicated by the `@` symbol. You create a new working change using `jj new` (by default as a child of the current change), and give it a description using `jj describe` (it starts out having an empty message). You can use `jj status` to get the metadata of the current change.

This much said, here's a quick demo of changing a file as part of a new change.

{{< rawhtml >}}
<script src="https://asciinema.org/a/XpE2I2f0bYp7EYudt1hH9lWlN.js" id="asciicast-XpE2I2f0bYp7EYudt1hH9lWlN" async="true"></script>
{{< /rawhtml >}}

`jj` on its own is an alias for `jj log`.

Additionally, most jj commands default to operating on the current working change, but let you operate on an arbitrary change (or set of changes!) via the `--revision / -r` flag.

## Pattern: git stash

This whole thing about the working copy being a Change may sound weird, but it brings with it an important feature - you can operate on the working copy like on any other change. Personally, I'm a frequent user of `git stash`. When I'm working on something, I often want to pause for a moment and work on/try something else, only to later come back to what I was working on (while leaving that other experiment as yet another stash).

In jj, I can just `jj new @-`, which will start a new change from the parent of my current working copy - leaving my working copy as a normal change, with all its file modifications "committed", so I can later come back to it via `jj edit`.

<demo>

<info box> the `-` after the `@` means "go one level up". It's basically "take the working copy change, and go to its parent". We'll come back to this briefly later, but suffice it to say that there's a whole sensible expression language here, and `@--` is similarly valid to refer to your grandparent change.

## Editing Changes in Weird Places

Another cool thing about changes is that you can freely edit any mutable change. You can also squish a change right after another (between it and its children). You can then keep modifying files in the context of that change, and any descendants will automatically be rebased on top of it (with their underlying commit SHAs likely changing, but change IDs staying the same). That sounds a bit scary if you're used to Git (and dealing with conflicts in the middle of a rebase) but fear not, it's actually pretty seamless, and we'll come back to it later.

You can pass `-A` and `-B` to `jj new` to indicate that you want to squish a new change after, or before, another change.

With this, when I notice a mistake in a change 3 levels back, I can just `jj edit <that-change-id>` (with my working copy remaining there for me to come back to), make a fix, which will auto-rebase all following changes (including my previous working change), and then `jj edit` to go back to my original working change.

<insert demo>

Occasionally you also need to rebase (move) a set of changes onto some change x, you can do that by using `jj rebase -s <change-id-to-rebase> -d <destination-change>`. The `-s` will bring all descendants along with the rebased change. There are other variations of this command for different scenarios.

## Pattern: Squishing a Fix Before the Current Change

You noticed a mistake in your last change, don’t want to fix it as part of this one, and instead want to squish a fix right before it, but you have already done some "uncommitted" work? Just do `jj new -A @-`. The `-A` means it’s squished after the previous change (referred to via `@-`), and before the current one (its child). After you make the fix, you can go back to the original working change via `jj edit @+` (`+` is the opposite of `-`). You could also make the fix in the original working change and run `jj split --interactive` to use your favorite diff ui to select what to push down to a separate change placed right before the current one.

## Bookmarks

In jj, instead of branches we have bookmarks. You create them using `jj bookmark create <name>` (abbreviated to `jj b c <name>`). You update a bookmark to point the current change using `jj bookmark set <name>`. You track remote bookmarks (the equivalent of checking out a remote branch, really) by doing `jj bookmark track <name>`.

When you add changes on top of a change that a bookmark is attached to, the bookmark won't automatically move to your new change (like a branch would), you have to `jj bookmark set` it manually. You can push bookmarks attached to the current change via `jj git push`. `jj git fetch` will fetch updates to bookmarks from your remote (so generally updates to branches).

## Pattern: Stacked PRs

A common use case with services like GitHub is to split up a big change into multiple PRs, let's say Multipart 1, Multipart 2 and Multipart 3 (from branches multipart-1, multipart-2, and multipart-3 respectively). Each of them is based on the previous one, so you effectively have the following linear graph, with bookmark pointers on the way:

<insert jj log output>

Now let's say Multipart 1 got reviewed and I need to fix something. In Git, once I add a commit to it (or modify an existing one), I would have to manually rebase all the other branches. Annoying!

How does jj help me here? I can just run `jj edit <buggy-change-id>` (TODO: insert correct change id), or `jj new -A <buggy-change-id>` (TODO: insert correct change id), make the fix, and run `jj git push -b "glob:multipart-*"`. Everything will be automatically rebased, and all the bookmarks will be updated and pushed. Effortless!

## Conflicts

One topic I haven't mentioned yet, and is surely by now giving you an unsettling feeling deep inside about all I've said before, are conflicts. With all this rebasing, that's **got** to become a pain, right? Well, it doesn't!

As opposed to Git, where conflicts kind of break your workflow, in the sense that you have to resolve them prior to doing anything else, jj handles conflicts in a first-class manner. A change can just "be conflicted". You can switch away from a conflicted change, you can create a new change on top of an existing conflicted change (and that will in turn also start out conflicted), you can edit a conflicted change, you can do anything. `jj status` and `jj log` will mention the conflict, but it won't block you.

<insert jj status and jj log conflict infos>

The conflict will be represented via conflict markers in the code:

<insert example code snippet>

and you can either manually resolve this conflict by editing the code itself, or use `jj resolve` to bring up your favorite three-way-merge tool (e.g. I've configured my jj to bring up a visual conflict resolver in Goland).

Once you fix the conflict in a change, all descendants of this change will also cease being conflicted. You could leave a change conflicted and only resolve the conflict in a follow-up child change - that is a completely valid and supported approach.

## Merges

Merges in jj are pretty boring - in a good way. You just create a change with multiple parents. E.g. `jj new x y z` will create a change with three parents. If there are any conflicts, you can resolve them as described above, either in the merge change itself, or in a follow-up change.

## Pattern: Working on Two Things at the Same Time

I've mentioned stacked PRs, but another situation you might have is working on two independent things in parallel. Let's say on bookmarks `thing-1` and `thing-2`.

In order to have both things simultaneosuly active in your codebase, you can create a development change via `jj new thing-1 thing-2 -m "dev"`, that will be a merge between both of them, but will stay local and you'll never push it (`-m` just lets you give a description to a change when creating it, without a separate `jj describe` invocation). You will however `jj edit` this development change to do your work.

<insert picture of jj log with the above workflow with multiple files>

Then, whenever you want to move some modifications into one of the branches, you can use `jj squash --into <target-change-id> <files>` to move all modifications to a set of files down into one of the branches. There's also `--interactive` where you can use a diff tool to choose modifications to squash into another change, and finally there's a [newer `jj absorb` command](https://jj-vcs.github.io/jj/latest/cli-reference/#jj-absorb) which can automate this process in certain scenarios.

<insert demo of the above>

In conclusion, you can keep working in a local-only merge-change of your branches, and selectively push down any modifications to the relevant branch (This setup would've seemed pretty scary before jj, right? I hope it's a bit less scary now.), and then push just those branches themselves.

## Revset Expressions

jj commands operate on revisions or sets of revisions (revsets). You can refer to those directly, or use a special expression language to describe them. You've seen me refer to a change previously via `@-`. That was a very simple expression that evaluated to the parent.

There is, however, much more. There are functions - like `parents(x)` to get the parents of a change - and operators - like `x+` to refer to the child of x, or `x::` for all descendants of x including x.

`jj log` accepts a revset expression, so you can use it to experiment with them. The default revset it displays is also configurable. Overall, the expression language is powerful and consistent, with simple things being generally easy, and harder things being (presumably, I've honestly spent too little time with it) possible.

See [this article](https://v5.chriskrycho.com/essays/jj-init/#revisions-and-revsets) for a much more extensive exploration of the jj expression language, and the [jj docs](https://jj-vcs.github.io/jj/latest/revsets/) themselves.

## Pattern: Partial Stashes

Another kind of stash I occasionally like to do is partial, where I temporarily roll back changes to a set of files to verify the before-after (e.g. confirm that a passing test was failing before).

In jj the split command works well for this. Just `jj split --parallel modifiedFile.txt` will move the file into a parallel change. You can do whatever you want to do, and later run `jj squash --from parallel_change_id` to get the file modifications back into the current change.

<demo with a Go function returning 42, a test checking that it's in fact 42, and an original state of the repo where the function is returning 41>

## Conclusion

I hope the above gave you an intuition for what Jujutsu, the VCS, is all about, and ideally even encouraged you to take a look at it.

If you'd like to do some more readings about jj, I've used the below articles and guides when learning it, and a lot of what I wrote above is inspired by parts of them. Check them out!

- https://v5.chriskrycho.com/essays/jj-init
- https://neugierig.org/software/blog/2024/12/jujutsu.html
- https://tonyfinn.com/blog/jj/
- https://v5.chriskrycho.com/journal/jujutsu-megamerges-and-jj-absorb/
