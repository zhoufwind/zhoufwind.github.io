---
layout: post
title: "Git Tutorial - III"
subtitle: "Git Tutorial: Using the Stash Command"
date: 2018-07-08 23:43:00 +0800
catalog: true
tags:
    - 开发
    - Git
---
# USING STASH FUNCTION

## BASIC USAGE

### Stash the change
```
$ echo "May the force be with you!" >> abc.txt
$ git diff
$ git stash save "Let's stash this change"
$ git diff
```

### Check the stash status
```
$ git stash list
```

### How to apply the stash?
1. Using apply command, make change back to file, but stash still there:
```
# Most recently change is 0, others +1
$ git stash apply stash@{0}
# Change back to previously
$ git checkout -- .
```

2. Grab top stash, apply the change, and drop the stash: 
```
$ git stash pop
```

### Remove the stash
1. Remove speicify stash:
```
$ git stash drop stash@{0}
```
2. Remove all the stash:
```
$ git stash clear
```

## USING STASH TO CHANGE BRANCH

```
$ git checkout master
$ echo "May the force be with you!" >> abc.txt
$ git status
$ git diff
$ git stash save "Make some change"
$ git checkout 'hello-there'
$ git stash pop
$ git diff
$ git add .
$ git commit -m "Make changes on feature branch"
```

### Reference
Git tutorial published at [YouTube][1] by [Corey Schafer][2] on Apr 17, 2015.

[1]: https://www.youtube.com/watch?v=KLEDKgMmbBI "Git Tutorial: Using the Stash Command"

[2]: http://coreyms.com/ "Corey Schafer"