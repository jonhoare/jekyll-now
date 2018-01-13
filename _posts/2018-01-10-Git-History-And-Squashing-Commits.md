You've been working on your own feature branch for a short while. You've been committing work regularly as you have been going. You've finished your work and you are about to create a Pull Request or merge directly into your master or develop branch. Do you really need or, more to the point, want to merge all of those commits into the your master or develop branch? I'm going to tell you why you should care about your history and how you can tell a story with it.

## Why should I care about my history?

Think about it, you've made several commits, you look at your git log since you forked your branch and you see something that resembles the following...
```git
afe7656 WIP Initial work
4a8efc2 WIP Fixed a typo
99fc547 Changes as required for spec
fe846f3 More changes
9475bd4 WIP End of day
74bce65 Steve's fixes
74ge67e Revert: Steve's fixes
bef6754 Finished work
```

Perhaps you work in a large team of developers, perhaps you have a separate DEVEOPS team that manages the build and deployments of your system.

You are about to merge all of these commits into your master/develop branch.

Do these commits really mean anything to anyone else apart from you? Heck, do they mean anything to you?

If you, or anyone else in your team, were to look back at these commits in a weeks time, a months time, or a couple of months time, say for example you have deployed these changes and it turns out the was an issue with the work you did and you need to fix up the commit, or even worse, you need to revert all the work you did. Would you be able to look back and find the commits that caused you a problem?

Believe me it is a very hard job to navigate history like this, even when it is your own work, let alone when it is someone on your DEVOPS team that needs to fix up the system.

Using source control like git to save your work in stages is fine and I highly encourage it, but just before you want to merge your branch or create a Pull Request, I suggest that you fix up this history so that it tells a story about the work you have done.

## How do I fix up my local history?

You have 2 options to fix up your local branch before you create a Pull Request or merge.

1. You can perform an interactive rebase.
2. You soft reset your HEAD at the common ancestor and re commit all your work.

### Option 1: Interactive Rebase

This is quite an advanced command, but quite easy to understand. You can find out about a rebase at my other [blog post]("/Git-Rebase-Vs-Merge").

An interactive rebase is exactly the same as a regular rebase, except that you can tell git what to do for each commit.

**Command:**
```git
git rebase master -i
```

When you run this command, git will rewind your HEAD back to the common ancestor commit between your branch and the master branch.

Git will then pull any new commits from master over to your branch and then finally, git will load up your default git editor, probably vim if you haven't set one explicitly, and list all of your branches commits to be applied and offer you several options of how you want to apply them.

For example you may be presented with the following based on our previous example.

```
pick afe7656 WIP Initial work
pick 4a8efc2 WIP Fixed a typo
pick 99fc547 Changes as required for spec
pick fe846f3 More changes
pick 9475bd4 WIP End of day
pick 74bce65 Steves fixes
pick 74ge67e Revert: Steves fixes
pick bef6754 Finished work

# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

You can see that each of the commits I made are listed and prefixed with **pick**. If we just save/close the editor without making any changes, then the rebase will continue as if we hadn't run the rebase with the `-i` option, as this will **pick** each commit and apply them as usual.

The command we are going to use is the **squash** command.

We will first **pick** the first commit and then **squash** all the other commits together.

***TIP:*** *You can also use the single letter prefix for the commands, s for squash and p for pick.*

```
p afe7656 WIP Initial work
s 4a8efc2 WIP Fixed a typo
s 99fc547 Changes as required for spec
s fe846f3 More changes
s 9475bd4 WIP End of day
s 74bce65 Steves fixes
s 74ge67e Revert: Steves fixes
s bef6754 Finished work
```

Now when we save and close the editor git will apply the first commit, and then for each further commit, the changes will be applied to the first commit and an editor will open, displaying the commit message from the first commit, and also each commits message commented out to remind you what the commit was. For each commit, you have the chance to change the commit message as required. For this I am going to change the commit message to a short, single line message, even with my JIRA tag, and maybe add a more optional description on a separated line.

```
JIR-1234 Added feature xyz

Here is a much more descriptive message about exactly what I did and how I did it.
```

When the rebase has finished you will need to force push your changes to the remote repository as you have rewritten your history compared to the remote.

Now when you look at your git log, you should see only a single commit which now nicely describes your work.

Once this commit goes in as a single commit you can easily revert this commit in the future, make changes to the commit, Cherry-Pick the commit if you need to.

***TIP:*** *You can also use the term `fixup` instead of squash during an interactive rebase. This performs the same action as the `squash` command, only it applies the changes without asking to change the commit message. You can therefore use the `reword` command for the first commit to reword this initial commit and then `fixup` all the other commits.*

### Option 2: Soft Reset HEAD

There is a small problem with option 1,using an interactive rebase to squash all your commits, and that is that assuming your branch has been around for a while, you have made all these commits, and master has moved on dramatically. Now when you try to perform a rebase on your branch with so many commits, if there are any merge conflicts it can be a real pain to resolve for each of your commits.

The even easier way to squash all of your commits to a single commit is to soft reset your branch and commit all your changes as a single commit.

First I need to find the common ancestor commit of my local branch and the master branch.

Command:
```git
git merge-base mybranch master
```

Result:
```
3d116ade8941ba44da14836d99c821f4ff583fe1
```

Now that I have this commit I need to soft reset my branch to this point in time.

A soft reset of a branch will reset the HEAD marker to the point in time you specify, but will keep all the changes made as un-staged changes. If you were to specify a hard reset instead, this would remove your changes.

```git
git reset --soft 3d116
```

Now my HEAD is back to the commit `3d116` and all of the changes I have made are all marked as un-staged. All I need to do now is add all the files to my staging area and then commit the changes as a single commit.

```git
git add -A
git commit
```

Now I get the chance to add my commit message much like I did with the interactive rebase.

```
JIR-1234 Added feature xyz

Here is a much more descriptive message about exactly what I did and how I did it.
```

Finally a force push to the remote repository to overwrite the history sees my nice single commit added to the remote. I am now ready to rebase the master or develop branch to get any changes before I merge back into master or develop, but this time if there are any conflicts, there is only one single commit that is going to have an issue for us to resolve.