---
layout: post
title: "Temporary Merge Branch in Diff3 Conflict Markers"
modified:
categories:
excerpt: "There's an edge case when using diff3 with git that can make conflicts
a nightmare to understand. Learn how to understand and resolve these most dreaded
conflicts."
tags: [git diff3 conflicts]
image:
  feature:
date: 2015-10-29T23:09:11-04:00
---

As one who uses git's diff3 style conflict markers, I have actually come to enjoy
performing git conflict resolution. Using diff3 makes conflicts significantly easier
to reason about. There's an edge case, however, that is not so understandable. You
may encounter something like this:

    <<<<<<< HEAD
        aaaaaa
    ||||||| merged common ancestors
    <<<<<<< Temporary merge branch 1
        bbbbbb
    =======
        cccccc
    >>>>>>> mybranch
        dddddd
    <<<<<<< HEAD
        eeeeee
    ||||||| merged common ancestors
        ffffff
    ||||||| merged common ancestors
        gggggg
    =======
    >>>>>>> Temporary merge branch 2
    =======
        hhhhhh
    >>>>>>> mybranch

What you're seeing in this example (with `Temporary merge branch` markers) is the
result of diff3 with a criss-cross merge conflict. I'll explain this with a sequence
of definitions.

# Definitions

* **merge base**: The commit where the two merging branches most recently diverged
  from. When a merge conflict occurs, different changes were made to the same lines
  in both branches. The *merge base* contains what those lines were before either
  branch changed them.
* **merged common ancestors**: diff3 outputs an additional "middle" section showing
  the lines as they were in the *merge base*. This is the starting point for both
  branches.
* **criss-cross merge**: A merge history where two branches merge into each other in
  ways that one could not have been a fast-forward merge. I give an example below. In
  a criss-cross merge situation, there are multiple *merge bases*.
* **Temporary merge branch**: When there are multiple *merge bases*, diff3 attempts
  to merge them together (using temporary merge branches) to form a single *common
  ancestor* to show in diff3's middle section. This works seamlessly when there are
  no conflicts, but when there are conflicts, you see the temporary merge branch's
  conflict markers inside the middle *merged common ancestors* section.

# Example criss-cross merge conflict scenario

A criss-cross merge occurs whenever two branches merge into each other at different
points in time.

    m3 *
       |\
       | \
       |  * B1
       |  |
    m2 *  * B0
       |\/|
       |/\|
    m1 *  * A
       | /
       |/
    m0 *

Consider this sequence of events:

* `m0` exists as origin/**m**aster
* I create a feature branch `feature-A` with one commit `A`
* `m1` gets committed to master by someone else
* I start a new feature branch `feature-B` that builds on `A`
* I merge `origin/master` (`m1`) into `feature-B`. It conflicts, and I resolve it.
  The merge commit is `B0`.
* I implement feature-B and commit the work as `B1`.
* `feature-A` is ready to ship, so someone merges it into `master`. It conflicts.
  They resolve it, but their resolution differs from the resolution in `B0`. The
  merge commit is `m2`.
* `feature-B` is ready to ship, so someone merges it into `master`. git tries to
  determine the *merge base*, but `m1` and `A` both qualify equally as merge bases.
  git merges `m1` and `A` in a *temporary merge branch*, which results in a conflict.
  We see diff3 output in the *merged common ancestors* section, similar to the OP's
  question.

# Reading the output

With diff3 off, this merge conflict would look simply like this:

    <<<<<<< HEAD
        aaaaaa
    =======
        hhhhhh
    >>>>>>> mybranch

First, with all the extra markers, you'll want to determine what the actual
conflicting lines are, so you can differentiate it from the diff3 common ancestor
output.

![conflict with merged common ancestor blurred][1]

aaaaaahhhhhh, that's a little better. ;-)

In the case where two conflict resolutions are conflicting, `aaaaaa` and `hhhhhh` are
the two resolutions.

Next, examine the content of the merged common ancestor.

![merged common ancestor conflicts, grouped][2]

With this particular merge history, there were more than 2 merge bases, which
required multiple temporary merge branches which were then merged together. The
result when there are many merge bases and conflicts can get pretty hairy and
difficult to read. Some say don't bother, just turn off diff3 for these situations.

Also be aware that git internally may decide to use different merge strategies to
auto-resolve conflicts, so the output can be hard to understand. Make sense out of it
if you can, but know that it was not intended for human consumption. In this case, a
conflict occurred when merging `mybranch` into `Temporary merge branch 1` between
`bbbbbb` and `cccccc`. Line `dddddd` had no conflicts between the temporary merge
branches. Then a separate conflict occurred when merging `Temporary merge branch 2`
into `HEAD`, with multiple common ancestors. `HEAD` had resolved the conflict by
merging `ffffff` and `gggggg` as `eeeeee`, but `Temporary merge branch 2` resolved
that same conflict by deleting (or moving) the line (thus no lines between `======`
and `Temporary merge branch 2`.

  [1]: http://i.stack.imgur.com/56OpC.png
  [2]: http://i.stack.imgur.com/xTKq4.png

How do you resolve a conflict like this? While technical analysis may be possible,
your safest option is usually to go back and review the history in all the involved
branches around the conflict, and manually craft a resolution based on your
understanding.

# Avoiding all this

These conflicts are the worst, but there are some behaviors that will help prevent
them.

1. *Avoid criss-cross merges*. In the example above, `feature-B` merged
   `origin/master` as `B0`. It's possible that this merge to stay up-to-date with
   master wasn't necessary (though sometimes it is). If `origin/master` was never
   merged into `feature-B`, there would have been no merge criss-cross, and `m3`
   would have been a normal conflict with `A` as the only merge base.

    <pre>
          m3 *              m3 *
             |\                |\
             | \               | \
             |  * B1           |  * B1
             |  |              |  |
          m2 *  * B0   VS   m2 *  |
             |\/|              |\ |
             |/\|              | \|
          m1 *  * A         m1 *  * A
             | /               | /
             |/                |/
          m0 *              m0 *
    </pre>

2. *Be consistent with conflict resolutions*. In the example, the *Temporary merge
   base conflict* only occurred because `m2` and `B0` had different conflict
   resolutions. If they had resolved the conflict identically, `m3` would have been a
   clean merge. Realize though that this is a simple criss-cross merge that ought to
   have had the same resolution. Other situations may rightly have different
   resolutions. Things get more complicated when there are more than 2 merge bases
   and multiple commits between the merge points. That said, if you are knowingly
   inconsistent with conflict resolutions in criss-cross situations, expect headaches
   later.
