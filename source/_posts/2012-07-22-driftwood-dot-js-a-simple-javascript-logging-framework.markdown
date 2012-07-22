---
layout: post
title: "Driftwood.js a simple javascript logging framework"
date: 2012-07-22 19:14
comments: true
categories: logging javascript error exception
---

We were joking around the [office](http://errplane.com) the other day and were wondering why there weren't any simple javascript logging libraries. The ones out there tried to completely copy log4j with abstract classes. We just wanted something that could have configurable log levels. Secondarily we needed to be able to post errors to the server.

Here are a few examples from Driftwood, all you need to do is set your environment and then start logging!

```
Driftwood.env("development"); 
Driftwood.debug("test"); 
```

If you want exceptions from users to be posted to your server just simply call the exception method. We automatically catch any window.onerrors and send them.

```
Driftwood.exception("exception text or object here")
```

So we cleaned up the internal code and added a nice suite of jasmine unit test. Now you can fork it on gitubhub [Driftwood.js](https://github.com/errplane/driftwood.js).

Also we have a slick looking documented source page [Driftwood source annotated](http://errplane.github.com/driftwood.js/)