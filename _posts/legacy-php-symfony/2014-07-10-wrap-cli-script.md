---
layout: post-en
category : "en"
title: How to wrap a legacy CLI script in Symfony app
tags: [development, php, symfony2, legacy, cookbook]
---
{% include JB/setup %}

> We have already examined how to wrap an arbitrary legacy URL in our Symfony app.
> Turns out, many apps also have some CLI (console) scripts, used to run cron jobs,
> or for some low-level configuration, or whatever.  What are we going to do with them?

Well, pretty much exactly what we did with URLs: instead of a catch-all `LegacyController`,
write a catch-all `LegacyCommand`.



