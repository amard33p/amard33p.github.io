---
title: "Resolving Git merge conflicts"
layout: post
excerpt: " "
tags:
  - git
---

## In an ideal world

I am about to add new feature or resolve some bugs. So I create a branch.  
`git checkout -b branch101`

I make the necessary code changes and commit. I then ensure that this local branch is at par with master.  
`git fetch origin`  
`git pull --rebase origin master`

```
 * branch            master     -> FETCH_HEAD
Already up to date.
```
`git push origin branch101` and submit Merge Request with master.

After MR is merged, ensure your local master is up-to-date  
`git checkout master`  
`git pull`

Optionally, delete feature/bug branch locally  
`git branch -d branch101`  
and on remote
`git push -d origin branch101`  
Or if branch was deleted directly in remote  
`git remote prune origin`

## In a not so ideal world

I follow the same steps as above but when I try to pull in master, I get a merge conflict.
```
$ git pull origin master --rebase
From https://somewhere.github.com/testrepo
 * branch            master     -> FETCH_HEAD
Auto-merging somefile.txt
CONFLICT (content): Merge conflict in somefile.txt
Automatic merge failed; fix conflicts and then commit the result.
```

I have a couple of ways to get out of this mess.

- Use `git mergetool` and attempt to resolve the conflicts in vimdiff/diffmerge or something similar.
- Or if the conflicts are too huge, I have to save my changes somewhere and start fresh with master's copy of the conflicting file.
  ```
  # First ensure your changes to the conflicting files are kept safely (in a separate branch/directory)
  # Keep the remote's (master) version of the file(s)
  git checkout --ours somefile.txt
  # For multiple files git checkout --ours .
  git add somefile.txt
  git commit -m "MERGE CONFLICT. Starting fresh with master's version of file"
  ```
  Note: master’s version = `--ours`, your branch’s version = `--theirs`  
  Now painstakingly add in your changes to the files, commit and push.


_References:_  
<https://shitalshah.com/p/git-workflow-branch-rebase-squash-merge/>  
<https://www.atlassian.com/git/tutorials/merging-vs-rebasing/>
