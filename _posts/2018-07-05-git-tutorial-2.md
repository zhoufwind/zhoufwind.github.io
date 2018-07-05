---
layout: post
title: "Git Tutorial - II"
subtitle: "Git Tutorial: Fixing Common Mistakes and Undoing Bad Commits"
date: 2018-06-30 12:51:00 +0800
catalog: true
tags:
    - 开发
    - Git
---
## FIXING MISTAKES

### Fixing mistakes before `add`
```
$ echo 'BALABALABA' >> abc.txt
$ git status
$ git diff
$ git checkout abc.txt
$ git status
```

### Fixing mistakes after `add` and `commit`
```
$ touch abc.txt
$ git add -A
$ git status
$ git commit -m "Commit wrong msg"
$ git log

# Commit message wrong, need correct.
$ git commit --amend -m "Commit correct msg"
```

### Adding changes in same commit
```
$ touch xyz.txt
$ git add xyz.txt
$ git commit --amend
$ git log
$ git log --stat
```

### Commit in wrong branch?
```
# commit in master branch by mistake, need 
# move to feature branch.
$ git log
$ git checkout hello-divide
$ git log
# hash-1 -> commit previously, second commit
$ git cherry-pick <hash-1>
$ git log
```

### Reset working directory
Three different type of resets:
- git reset --soft
- git reset --mixed(default)
- git reset --hard

#### soft reset - staged changes after reset
Reset back to the commit we specifid, but will keep our changes that we've made in the staging directory(AFTER ADD).
> Changes to be committed
```
$ git checkout master
$ git log
# hash-0 -> commit initialized, first commit
$ git reset --soft <hash-0>
# Now we no longer have second commit
$ git log
# But still have changes we've made, staged
$ git status
```

#### mixed reset - Unstaged changes after reset
Reset back to the commit we specifid, keep the changes, however, the changes in stating area, in the working directory.
> Changes not staged for commit
> Untracked files
```
# without parsing any keywords, using mixed
$ git reset <hash-0>
# unstages
$ git status
```

#### hard reset - back to initial commit
Reverse all the track file back to state that they were, but leaves any untracked files alone.
> Untracked files
```
$ git reset --hard <hash-0>
# back to initial
$ git status
```

#### remove untracked file
It will save your day when you unzip many files into working directory.
> nothing to commit, working directory clean
```
$ git clean -df
```

### Garbage collect
Grab the hash before the reset, and copy that in a detached HEAD state.
It's a livesaver if you had lost some critical files that you really didn't mean to delete or that if you accidently did a reset on something.
```
$ git reflog
# hash-x -> before the reset
$ git checkout <hash-x>
$ git branch backup
$ git branch
$ git checkout master
$ git branch
$ git checkout backup
$ git log
$ git reflog
```

### Revese after push

```
$ git revert <hash-x>
# history have been logged
$ git log
$ git diff 
```

### Reference
Git tutorial published at [YouTube][1] by [Corey Schafer][2] on Aug 3, 2015.

[1]: https://www.youtube.com/watch?v=FdZecVxzJbk "Git Tutorial: Fixing Common Mistakes and Undoing Bad Commits"

[2]: https://www.youtube.com/channel/UCCezIgC97PvUuR4_gbFUs5g "Corey Schafer"