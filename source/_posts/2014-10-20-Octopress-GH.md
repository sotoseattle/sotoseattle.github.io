---
layout: post
title: "Installing Octopress at github.io"
date: 2014-10-20 11:00
comments: true
categories: GitHub, Octopress
---

I am collecting posts from my previous blog, ideogrammatic.com, adding some new ones I have written over the last month and putting it all together in github.io as my personal blog.

I am using Octopress with a tweaked mnml theme and have got hold of a nice domain with my handle sotoseattle to point to.

The basic deployment of an Octopress blog into your github.io is easy, and there are many posts with the details in the Internet so I am assuming you have gone through the paces.

Now, if you are like me, you'll hit problems in unexpected places. I have found that it helps to understand how Octopress and GitHub actually use git repos between them.

In essence you'll end up using two different git repos inside both your Octopress code base. And the fun part is that both point to the same github repo!

1.- the first can be found inside the folder _deploy:

* ```rake generate``` compiles the site from all the files in the source directory (and other resources) and places everything inside the _deploy folder.
* ```rake deploy``` pushes it (the files inside _deploy) up to GitHub, actually to the to origin/master branch.

So, for example, if we have a custom domain and we want it to point to our github.io site, we need a CNAME file that we place inside the source folder. After rake generate and rake deploy, it will show up in the root of origin/master (and inside our _deploy folder)

2.- The second git repo, you create it on the root:

For this one we create a single branch, "source", that holds everything (all files, source and _deploy) and we push only to origin/source. Aha!, this repo goes to the 'source' branch in github, the generated one goes to 'master'. So we don't have a master branch in this local git repo, only source, and in this way we avoid pushing it to master in origin (which already tracks the first repo!).

Messy explanation, I know, but once you wrap your mind around the concept of 2 different repos pointing to 2 branches of the same upstream repo, it all makes sense.

Now, I may be off base in this explanation, if so, please set me straight in the comments.

Good luck with the Octos!

{% img center /images/Oct14/octocat_octopress.png 400 Code Fellows logo %}
