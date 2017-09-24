---
layout: post
title: "Take the pain out of git conflict resolution: use diff3"
modified:
categories:
excerpt: "Hating git conflicts? Use git's built-in diff3 option to make sense out of it all!"
tags: [git diff3 conflicts]
comments: true
image:
  feature:
date: 2017-09-23T22:15:00-04:00
---

> +1 for telling me about diff3 alone—how often I was looking on an
> incomprehensible conflict, cursing whoever is responsible for not telling me
> what the common ancestor had to say. Thank you very much.

Well said, [John][] of StackOverflow. The `diff3` conflict resolution strategy
is a hidden gem in git that has saved me uncountable conflict-resolution
headaches and has turned git conflict resolution into something of a joy for me.
Weird, I know!

This post is an adaptation from a [StackOverflow answer] that I wrote in 2012
that deserves to live on its own.

[StackOverflow answer]: https://stackoverflow.com/a/11219380/87298
[John]: https://stackoverflow.com/questions/457927/git-workflow-and-rebase-vs-merge-questions#comment16388231_11219380

---

Example
-------

Let's take a look at this conflict. How would you resolve it?

    <<<<<<< HEAD
    GreenMessage.send(include_signature: true)
    =======
    BlueMessage.send(include_signature: false)
    >>>>>>> merged-branch

Looking at the conflict alone, it's impossible to tell what each branch changed
or what its intent was. This lack of information is the biggest cause that I see
that makes conflict resolutions frustrating and incorrect.

diff3 to the rescue!

Turn this thing on!
-------------------

    git config --global merge.conflictstyle diff3

After running this command to turn on diff3, each new conflict will have a 3rd
section, the merged common ancestor.

How do I read this?
-------------------

    <<<<<<< HEAD
    GreenMessage.send(include_signature: true)
    ||||||| merged common ancestor
    BlueMessage.send(include_signature: true)
    =======
    BlueMessage.send(include_signature: false)
    >>>>>>> merged-branch

The **merged common ancestor** section shows what the conflicting line(s) were
before either of the merged branches changed them. This is our baseline frame of
reference. You'll find the merged common ancestor code for this example on the
center line between `||||||| merged common ancestor` and `=======`

The code between `<<<<<<< HEAD` and `||||||| merged common ancestor` shows what
the code looks like at HEAD, the current commit (for rebase, this will be the
rebase target).

The code between `=======` and `>>>>>>> merged-branch` shows what the code looks
like at `merged-branch`—the branch or commit being merged.

Note that any of the section markers may have 0 lines between them, indicating
an empty section. I'll cover this more in the examples below.

The Conflict Resolution Pattern
-------------------------------

With diff3 or without, the pattern of merging code changes remains the same:

1. Compare the merged common ancestor with each of the conflicting changes.
2. Choose one change as your starting point (usually the more complex change)
   and one change to apply (usually the simpler change).
3. Apply the simpler change to the other change (merge the changes).

Let's try this with our example.

First, compare each change (HEAD, and then merged-branch) with the merged common
ancestor, trying to determine each change's intent. HEAD changes `BlueMessage`
to `GreenMessage`. Its intent is to change the class used to GreenMessage,
passing the same parameters. `merged-branch` changes `include_signature` from
`true` to `false`, intending to stop inclduing a signature in messages.

In this case, both changes are simple, so I'll arbitrarily start with the code
in HEAD, and apply the other change: `include_signature` changes from `true` to
`false`.

After removing conflict section markers and everything else other than the
merged code, we have this:

    GreenMessage.send(include_signature: false)

Repeat for each conflicting hunk, and we're done! Add the resulting merged
files and continue with the merge/rebase/cherry-pick.

More examples, with explanations
--------------------------------

### Lines added to the same location; order ambiguous

    <<<<<<< HEAD
    @import 'some_file';
    ||||||| merged common ancestor
    =======
    @import 'other_file';
    >>>>>>> merged-branch

When the merged common ancestor is a blank section, such as this, each branch
added lines at the same position. In this case, we typically want to keep both
lines that were added, ordered in whichever order makes the most sense.

Example resolution:

    @import 'some_file';
    @import 'other_file';

### Changes with the same intent

    <<<<<<< HEAD
    # Broken; see #35
    ||||||| merged common ancestor
    =======
    # this doesn't work
    >>>>>>> merged-branch

Here again, each branch added lines at the same position. However in this case,
both changes share a common intent: document breakage in the surrounding code.
This type of conflict can also happen when multiple people fix the same bug in
different ways. With the same intent, keeping both additions would be redundant
here and harmful in other scenarios. Instead, we'll pick the implementation that
best accomplishes the intent, or combine the best of both.

Example resolution:

    # Broken; see #35

### Moved or deleted lines

    <<<<<<< HEAD
    ||||||| merged common ancestor
    BlueMessage.send(include_signature: true)
    =======
    BlueMessage.send(include_signature: false)
    >>>>>>> merged-branch

When one of the branch's sections is empty, this indicates that the lines were
deleted or moved in that branch. In this case, either a message is no longer
being sent, or the call to `BlueMessage.send` has moved elsewhere. You must
determine which. If the call was deleted, then `merged-branch`'s change may
have become irrelevant. If the change in HEAD moved the call elsewhere or
changed how it was done, then we must find it and apply the `include_signature`
change if it's still relevant.

The resolution here is likely to delete this conflict block, and search for
where we may need to apply the `include_signature` change elsewhere.
