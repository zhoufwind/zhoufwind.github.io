---
layout: post
title: "Git Tutorial"
subtitle: "Git Tutorial for Beginners: Command-Line Fundamentals"
date: 2018-06-30 12:51:00 +0800
catalog: true
tags:
    - 开发
    - Git
---
### Installing git
Go to [git-scm.com](https://git-scm.com/)

### First time setup

#### Check version
```
$ git --version
git version 2.17.1
```

#### Set config values
```
$ git config --global user.name "Feng Zhou"
$ git config --global user.email "zhoufwind@hotmail.com"

$ git config --list
```

#### Help
- git help \<verb\>
- git \<verb\> --help
```
$ git help config
$ git config --help
$ git add --help
```

### Getting started
Two common scenarios.

#### Initialize a repository from existing code
```
$ git init
```

#### Before first commit
```
$ git status
```

#### Add gitignore file
```
$ touch .gitignore
$ cat .gitignore
.DS_Store
.project
*.pyc
```

#### Git staging
![img](/img/in-post/post-git-tutorial/git-staging.jpeg)

#### ADD FILES TO STAGING AREA
```
$ git add calc.py
$ git add -A
$ git status
```

#### REMOVE FILES FROM STAGING AREA
```
$ git reset calc.py
$ git reset
$ git status
```

#### First commit
```
$ git add -A
$ git commit -m "Initial commit"
$ git status
$ git log
```

#### Cloning a remote repo
```
$ git clone <url> <where to clone>

$ git clone ../remote_repo.git .

$ git clone https://github.com/zhoufwind/Hello-World.git .
```

#### Viewing information about the remote repository
```
$ git remote -v
$ git branch -a
```

#### Pushing changes
```
$ git diff
$ git status
$ git add .
$ git commit -m "Make a change"

$ git pull origin master
$ git push origin master
```

### Common workflow

#### Create a branch for desired feature
```
$ git branch hello-divide

$ git branch
  hello-divide
* master

$ git checkout hello-divide
Switched to branch 'hello-divide'

$ git branch
* hello-divide
  master

$ cp ~/Temp/hello.sh .
$ git status
$ git add -A
$ git commit -m "Adding demo script"
```

#### After commit push branch to remote
```
$ git push -u origin hello-divide
$ git branch -a
```

#### Merge a branch
```
$ git checkout master

$ git pull origin master

$ git branch --merged

$ git merge hello-divide

$ git push origin master
```

#### Deleting a branch
```
$ git branch --merged

$ git branch -d hello-divide

$ git branch -a

$ git push origin --delete hello-divide
```

### Reference
Git tutorial published at [YouTube][1] by [Corey Schafer][2] on Aug 3, 2015.

[1]: https://www.youtube.com/watch?v=HVsySz-h9r4 "Git Tutorial for Beginners: Command-Line Fundamentals"

[2]: https://www.youtube.com/channel/UCCezIgC97PvUuR4_gbFUs5g "Corey Schafer"