I finally understand it:

There are two different git repos both pointing to the same github repo:

1.- the first inside _deploy:

>> * rake generate greates the site in _deploy (from all other stuff)
>> * rake deploy push it up (only this _deploy) to origin/master

So, to include the CNAME file in the _deploy folder we need to place it first in source. Then, the generate action will place in _deploy.

2.- The second one in root:

We have created a single branch, "source", that holds everything and we push only to origin/source. So we don't have a master branch, to avoid pushing it to master in origin (which already tracks the first repo!).

