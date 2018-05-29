---
layout: post
title:  "Command prompt setup"
date:   2018-05-29 18:31:00 +0200
categories: command prompt, git branch
---
Quite for some time I was taking command prompt for granted, thinking that it is just as it is. But today is the day, I'm setting up the command prompt that I like!

First thing to do was actually to decide what exact command prompt I like. After searching the Internet for details about setting up prompt, I have picked two links to put in this post: [first][ps1-howto] about general details for PS1 environment variable, and [second][show-branch] about putting current git branch to command prompt. For me it appears that ideal prompt is as easy as `<username>@[<directory> (:<git branch, if any>)] $ `. My final setup is published at github as a [gist][ps1-gist].

In short, this is how to detect current git branch (here I put prefix ` :` before branch name in order to completely remove branch section of command prompt if there aren't any git branch - this happens, sometimes I don't go to folders between git repositories, that have no git branch).

```bash
## Get output of 'git branch' and leave only row that contains '*'
git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ :\1/'
```

From [article][ps1-howto] I can find needed characters for username and current directory.

```bash
## \u - the username of the current user
## \w - the current working directory, with $HOME abbreviated with a tilde
```

Now it's time to put everything together and get final result. Put this into `~/.bashrc` on Linux and/or `~/.bash_profile` on MacOS.

```bash
get_git_branch() {
  git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ :\1/'
}

export PS1='\u@[\w$(get_git_branch)] $ '
```

I think I've never been typing words 'command prompt' so much before writing this post.

[ps1-howto]: https://www.cyberciti.biz/tips/howto-linux-unix-bash-shell-setup-prompt.html
[show-branch]: https://gist.github.com/githubteacher/e75edf29d76571f8cc6c
[ps1-gist]: https://gist.github.com/sergey-melnychuk/89e6e78ac212dd4b55cc819356108459
