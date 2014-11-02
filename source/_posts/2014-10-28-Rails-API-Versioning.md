---
layout: post
title: "Rails Api Versioning"
date: 2014-10-28 16:17
comments: true
categories: Rails, API
---

{% img right /images/Oct14/versionapi.png 200 RailsCast %}

Rails provide a powerful framework to build your own API. But as you evolve your API will evolve too, your code will change and even your schema will mutate. That is OK as long as you have a rational way to structure the different versions of the api, how they are called and what they return.

There are different ways to achieve this versioning, I will be enumerating the pros and cons of three of them.

### Pre-pend (the Ryan Bates Way)

[RailsCast](http://railscasts.com), the ultimate resource for all things rails, goes into specific details of how to manually set up your own versioning system in its [episode 350](http://railscasts.com/episodes/350-rest-api-versioning?view=asciicast).

It involves modifying your routes so your apis resources are encapsulated in namespaces. There are many advantages to this approach, first and foremost it forces you to be aware of what is happening, you learn your tool in the process and you don't depend on external code (versioning gem).

On the other hand, you'll end up with parallel versions of the same code living side by side, not the most DRY way to keep your code base. Furthermore, you are responsible for tracking changes to your schema, and then adapting and maintaining each of the version calls.

<!--more-->

Nevertheless, this is the approach I have generally used since it provides me with the most experience and learning.

You can actually have it both ways, pre-pending the api version and automating with a gem (up to a point) with the Versionist Gem, which provides three different ways to version your api out of the box:

- Specifying version via an HTTP header
- Specifying version by pre-pending paths with a version slug
- Specifying version via a request parameter

Pre-pending the path with a version slug is the same as the method discussed in RailsCast, visual, fast and intuitive. The problem is that the gem won't move files for you so you'll need to re-organize some files around and change some configuration details.


### Request Header

The [Versionist](https://github.com/bploetz/versionist) gem also allows us to include the version to use as a parameter in the header of the request call. Which has the benefits and disadvantages of hiding away this information in the header. It is less transparent and user friendly, but great for automated calls. Less opportunity to fudge at the cost of a higher initial cost of setting the system up.


### Url Parameter

The third way Versionist can work is by appending the api version as a parameter at the end of the call, which although benefiting from being as somewhat visual, it fudges the url mixing resource parameters with unrelated information. This practice is frowned upon in the rails development community.

### MIME Type

You could register a MIME type for each api version and use the rails' respond_to handle the response. Check Robbie Clutton [post about it](http://pivotallabs.com/api-versioning/).

The registration happens in the config/initializers/mime_types.rb, and all you need is to reconfigure your controllers.

### API Model

Gems like [ApiVersioning](https://github.com/craigs/api_versioning) use a model-based approach. Where a controller renamed as a `presenter`, handles API responses. It allows to specified the version in the header or passed as a parameter.

Thanks to Scot Hale who pointed me in the right direction in order to research this topic!
