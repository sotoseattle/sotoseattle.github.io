---
layout: post
title: "Ask before pushing"
date: 2014-09-16 8:01
comments: true
categories: Git
---

During the usual git work flow we get so used to re-keying commands that sometimes we push up without thinking. Most things are easy to fit locally in git, besides, as they say, when in doubt reset --hard. But once your code is committed up to a publicly available and centralized repo, you become the butt of jokes in the office, people point at you behind your back and changing history becomes a nightmare.

I have found a little trick that helps with this, git hooks!

Inside your .git folder there is another one, 'hooks', and inside, there is a treasure trove of configurable callbacks to personalize your git work flow.

There are many good sources in the Internet to check. Here is a good walk through: [link](http://blog.ittybittyapps.com/blog/2013/09/03/git-pre-push/). And this is the [Chacon bible](http://git-scm.com/book/en/Customizing-Git-Git-Hooks) chapter about it.

I ended including the following file '.git/hooks/pre-push', which forces a confirmation every time I intend to push to master. Many other variants are possible, like making sure that your tests have run before pushing, I recommend exploring them all.

```bash
#!/bin/bash

protected_branch='master'
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

if [ $protected_branch = $current_branch ]
then
    read -p "You're about to push master, is that what you intended? [y|n] " -n 1 -r < /dev/tty
    echo
    if echo $REPLY | grep -E '^[Yy]$' > /dev/null
    then
        exit 0 # push will execute
    fi
    exit 1 # push will not execute
else
    exit 0 # push will execute
fi
```
