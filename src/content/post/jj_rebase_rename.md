---
title: "Rebasing renamed files with jj"
publishDate: "2025-05-23T14:36:00-05:00"
draft: false
description: How to resolve nasty merge conflicts
tags: ["jj", "git", "vcs"]
---

Let's say you have a branch that modifies files `A`, `B`, and `C`.
You want to rebase your branch on the latest version of main/master, but you discover that those files have been renamed.
`A` is now `D`, `B` is `E`, and `C` is `F`.
You hope that jj/git will be smart enough to know it just needs to apply your diffs to a different file now, but you are horrified to discover that it doesn't.
Instead, jj complains that the rev is trying to edit a file that doesn't exist, so your conflict diff is just the entire state of the file at that rev, which isn't super helpful.
If your changes are minimal, then it's not too hard to just manually transfer them over to the new files, but what if you had 20+ revs with a significant number of changes?
Manually copying the changes to the new file is going to take a long time and there's a good chance you'll miss something.

I ran into this exact scenario the other day.
The best part was that I was the one who renamed the files on master, so I put myself in this situation, naïvely thinking that it would be easy to resolve the conflicts.
It took me hours to come up with a solution, but I'm pretty happy with it.

## Solution

The basic idea is that we simply rename the files in each of our revs.
However, this isn't as easy as it sounds.

If you rename the files in the first rev of your branch, all of the later revs will have conflicts.
The issue there is that if you rename the files in a rev with conflicts, the conflicts will just get moved over to the new file, but the conflicts won't be resolved.

So what if you instead work from the head of the branch down to the root of the branch?
Once you rename the files in the second rev from the top, the head rev will have conflicts, since both revs rename the files.

So this seems almost impossible without adding a new feature to jj.
However, it turns out it is possible, it just requires a little extra work.

### Step 1 - Renaming script

You'll need to create a script that renames the files in your repo correctly.
Don't worry about updating any paths inside of the files, you just need to `mv` the file to the new location.
This script should live outside of your repo so you have access to it across all revs.

```bash
# rename-files.sh

mv A D
mv B E
mv C F
```


### Step 2 - Duplicate the branch

This is the magic right here.
We're going to duplicate the branch, but with updated filenames.
This will produce revs with the exact same diffs as the original branch, but the diffs will point to the new filenames.

You'll start with the first commit on your branch and work your way towards the head.
For each rev, you'll create a new rev on your new branch, `jj restore --from <old_rev>`, then run your renaming script.

It is important to note that you need a rev at the root of your new branch that only performs the renames.
You'll see why this is needed later.

Let's say your `jj log` looks like:

    @  xyvkxnlu ewatson@example.com 2025-05-23 09:43:32 my-branch 5564ad65
    │  Edit A & C
    ○  uuxpkymp ewatson@example.com 2025-05-23 09:43:15 793a223d
    │  Edit B & C
    ○  nozrrvkk ewatson@example.com 2025-05-23 09:42:29 ccbb0709
    │  Edit A & B
    ○  tsokkvqk ewatson@example.com 2025-05-23 09:41:30 old-main e5aa0c4e

First, we'll branch off of `old-main` and rename the files.
This rev is unlike the others we'll make since it isn't correlated with any rev in the original branch.

    $ jj new t
    Working copy  (@) now at: nwkxvunp 261d97db (empty) (no description set)
    Parent commit (@-)      : tsokkvqk 9d9a3630 old-main | (no description set)
    Added 0 files, modified 3 files, removed 0 files
    
    $ rename-files.sh
    
    $ jj st
    Working copy changes:
    R {A => D}
    R {B => E}
    R {C => F}
    Working copy  (@) : nwkxvunp 575b1efe (no description set)
    Parent commit (@-): tsokkvqk 9d9a3630 old-main | (no description set)
    
    $ jj log
    @  nwkxvunp ewatson@example.com 2025-05-23 09:59:16 575b1efe
    │  (no description set)
    │ ○  xyvkxnlu ewatson@example.com 2025-05-23 09:59:02 my-branch 62b239b9
    │ │  Edit A & C
    │ ○  uuxpkymp ewatson@example.com 2025-05-23 09:58:41 8882936f
    │ │  Edit B & C
    │ ○  nozrrvkk ewatson@example.com 2025-05-23 09:58:20 04ccd9a9
    ├─╯  Edit A & B
    ○  tsokkvqk ewatson@example.com 2025-05-23 09:57:54 old-main 9d9a3630

Next, create a new rev, `jj restore` from the first rev on the original branch, and run your renaming script.

    $ jj new
    Working copy  (@) now at: wskxplyw 6313015d (empty) (no description set)
    Parent commit (@-)      : nwkxvunp 575b1efe (no description set)
    
    $ jj restore --from no
    Working copy  (@) now at: wskxplyw f69e74a2 (no description set)
    Parent commit (@-)      : nwkxvunp 575b1efe (no description set)
    Added 3 files, modified 0 files, removed 3 files
    
    $ jj st
    Working copy changes:
    A A
    A B
    R {F => C}
    D D
    D E
    Working copy  (@) : wskxplyw f69e74a2 (no description set)
    Parent commit (@-): nwkxvunp 575b1efe (no description set)
    
    $ rename-files.sh
    
    $ jj st
    Working copy changes:
    M D
    M E
    Working copy  (@) : wskxplyw 33e721b0 (no description set)
    Parent commit (@-): nwkxvunp 575b1efe (no description set)
    
    $ jj log
    @  wskxplyw ewatson@example.com 2025-05-23 10:05:15 33e721b0
    │  (no description set)
    ○  nwkxvunp ewatson@example.com 2025-05-23 09:59:16 575b1efe
    │  (no description set)
    │ ○  xyvkxnlu ewatson@example.com 2025-05-23 09:59:02 my-branch 62b239b9
    │ │  Edit A & C
    │ ○  uuxpkymp ewatson@example.com 2025-05-23 09:58:41 8882936f
    │ │  Edit B & C
    │ ○  nozrrvkk ewatson@example.com 2025-05-23 09:58:20 04ccd9a9
    ├─╯  Edit A & B
    ○  tsokkvqk ewatson@example.com 2025-05-23 09:57:54 old-main 9d9a3630

You can copy over the description using the following command (assuming you're using bash/zsh/etc).
This isn't necessary, but makes it easier to figure out which revs go together.
Note that you'll have to replace `no` with the ID of the original rev.

    $ jj desc -m "$(jj log --no-graph -r no -T 'description')"
    
    Working copy  (@) now at: wskxplyw 8980e953 Edit A & B
    Parent commit (@-)      : nwkxvunp 575b1efe (no description set)
    
    $ jj log
    @  wskxplyw ewatson@example.com 2025-05-23 10:07:39 8980e953
    │  Edit A & B
    ○  nwkxvunp ewatson@example.com 2025-05-23 09:59:16 575b1efe
    │  (no description set)
    │ ○  xyvkxnlu ewatson@example.com 2025-05-23 09:59:02 my-branch 62b239b9
    │ │  Edit A & C
    │ ○  uuxpkymp ewatson@example.com 2025-05-23 09:58:41 8882936f
    │ │  Edit B & C
    │ ○  nozrrvkk ewatson@example.com 2025-05-23 09:58:20 04ccd9a9
    ├─╯  Edit A & B
    ○  tsokkvqk ewatson@example.com 2025-05-23 09:57:54 old-main 9d9a3630

Now we repeat the same steps for the next rev.

    $ jj new
    Working copy  (@) now at: srqvwpll 671820eb (empty) (no description set)
    Parent commit (@-)      : wskxplyw 8980e953 Edit A & B
    
    $ jj restore --from u
    Working copy  (@) now at: srqvwpll 77da061f (no description set)
    Parent commit (@-)      : wskxplyw 8980e953 Edit A & B
    Added 3 files, modified 0 files, removed 3 files
    
    $ rename-files.sh
    
    $ jj desc -m "$(jj log --no-graph -r u -T 'description')"
    
    Working copy  (@) now at: srqvwpll c3f5ee3f Edit B & C
    Parent commit (@-)      : wskxplyw 8980e953 Edit A & B
    
    $ jj st
    Working copy changes:
    M E
    M F
    Working copy  (@) : srqvwpll c3f5ee3f Edit B & C
    Parent commit (@-): wskxplyw 8980e953 Edit A & B
    
    $ jj log
    @  srqvwpll ewatson@example.com 2025-05-23 10:11:25 c3f5ee3f
    │  Edit B & C
    ○  wskxplyw ewatson@example.com 2025-05-23 10:07:39 8980e953
    │  Edit A & B
    ○  nwkxvunp ewatson@example.com 2025-05-23 09:59:16 575b1efe
    │  (no description set)
    │ ○  xyvkxnlu ewatson@example.com 2025-05-23 09:59:02 my-branch 62b239b9
    │ │  Edit A & C
    │ ○  uuxpkymp ewatson@example.com 2025-05-23 09:58:41 8882936f
    │ │  Edit B & C
    │ ○  nozrrvkk ewatson@example.com 2025-05-23 09:58:20 04ccd9a9
    ├─╯  Edit A & B
    ○  tsokkvqk ewatson@example.com 2025-05-23 09:57:54 old-main 9d9a3630

And again for the final rev.

    $ jj new
    Working copy  (@) now at: lkwrmmxl 458d5336 (empty) (no description set)
    Parent commit (@-)      : srqvwpll c3f5ee3f Edit B & C
    
    $ jj restore --from x
    Working copy  (@) now at: lkwrmmxl 2305af1f (no description set)
    Parent commit (@-)      : srqvwpll c3f5ee3f Edit B & C
    Added 3 files, modified 0 files, removed 3 files
    
    $ rename-files.sh
    
    $ jj desc -m "$(jj log --no-graph -r x -T 'description')"
    
    Working copy  (@) now at: lkwrmmxl 99d2177e Edit A & C
    Parent commit (@-)      : srqvwpll c3f5ee3f Edit B & C
    
    $ jj st
    Working copy changes:
    M D
    M F
    Working copy  (@) : lkwrmmxl 99d2177e Edit A & C
    Parent commit (@-): srqvwpll c3f5ee3f Edit B & C
    
    $ jj log
    @  lkwrmmxl ewatson@example.com 2025-05-23 10:12:54 99d2177e
    │  Edit A & C
    ○  srqvwpll ewatson@example.com 2025-05-23 10:11:25 c3f5ee3f
    │  Edit B & C
    ○  wskxplyw ewatson@example.com 2025-05-23 10:07:39 8980e953
    │  Edit A & B
    ○  nwkxvunp ewatson@example.com 2025-05-23 09:59:16 575b1efe
    │  (no description set)
    │ ○  xyvkxnlu ewatson@example.com 2025-05-23 09:59:02 my-branch 62b239b9
    │ │  Edit A & C
    │ ○  uuxpkymp ewatson@example.com 2025-05-23 09:58:41 8882936f
    │ │  Edit B & C
    │ ○  nozrrvkk ewatson@example.com 2025-05-23 09:58:20 04ccd9a9
    ├─╯  Edit A & B
    ○  tsokkvqk ewatson@example.com 2025-05-23 09:57:54 old-main 9d9a3630

So now we have a new branch that is almost identical to the original branch, but has the new filenames.
And importantly, the actual rename operation occurs in `nwkxvunp`, which contains no other changes.




### Step 3 - Rebase the new branch

Now you'll want to rebase the new branch on the latest version of main/master.
You'll notice that you get conflicts again!
Have we just wasted a bunch of time?
Fear not, just `jj abandon` that extra rev (e.g. `nwkxvunp`) we had at the beginning of the new branch, the one that just has renames and no other changes.
Suddenly all the conflicts related to renamed files will go away and you'll just be left with the true conflicts you care about.




### Step 4 - Resolve the true conflicts

Now you'll just resolve the remaining conflicts on the new branch, which are the conflicts that would have been there even if the files weren't renamed.




### Step 5 - Rebase the old branch

Rebase the old branch onto the latest version of main/master.
Ignore the conflicts, we're about to fix that.




### Step 6 - Copy the new branch back to the original branch

For each rev on the original branch, starting at the root and going to the head, execute `jj restore --from <new_rev_id>`.
This copies the contents of the news rev into the original revs.

So for the earlier example:

    ○  xyvkxnlu ewatson@example.com 2025-05-23 09:59:02 my-branch 62b239b9
    │  Edit A & C
    ○  uuxpkymp ewatson@example.com 2025-05-23 09:58:41 8882936f
    │  Edit B & C
    @  nozrrvkk ewatson@example.com 2025-05-23 09:58:20 04ccd9a9
    │  Edit A & B
    │ ○  lkwrmmxl ewatson@example.com 2025-05-23 10:12:54 new-branch 99d2177e
    │ │  Edit A & C
    │ ○  srqvwpll ewatson@example.com 2025-05-23 10:11:25 c3f5ee3f
    │ │  Edit B & C
    │ ○  wskxplyw ewatson@example.com 2025-05-23 10:07:39 8980e953
    ├─╯  Edit A & B

You would execute:

    jj edit nozrrvkk
    jj restore --from wskxplyw
    
    jj edit uuxpkymp
    jj restore --from srqvwpll
    
    jj edit xyvkxnlu
    jj restore --from lkwrmmxl

Make sure you are doing this in the right direction.
We want to overwrite the original revs with the contents of the new revs, not vice-versa.




### Done!

You can delete the new branch now if you want.
Our original branch is caught up with main/master and all the conflicts have been resolved.

