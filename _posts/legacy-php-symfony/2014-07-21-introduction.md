---
layout: post-en
category : "en"
title: "Symfony and Legacy Code: Introduction"
tags: [php, symfony2, symfony-legacy]
---
{% include JB/setup %}

There are billions of lines of horrible code in the world.  There are
billions (I think) of lines of PHP code in the world.  These two sets
have significant overlap.

PHP, as of 2014, remains the most widely used web development language,
particularly for applications which do not warrant use of Java or .NET or
something else "serious".  At the same time, PHP is widely considered
[one of the worst languages][php-fractal] there is.  Now, "worst", I believe, is an
overstatement; by itself PHP is merely mediocre.  But its low barrier
for entry, together with steadily high demand for web development jobs,
resulted in proliferation of messy PHP code.

Now, it is certainly possible to write good PHP.  [Symfony 2][symfony],
the best, in my opinion, PHP web framework at the moment, is a fine
example.  PHP written in Symfony style tends to look a lot like Java.
Still not glamorous, but much easier and more pleasant to work with.

> The sad truth is, most of PHP code in the world is not written with
> Symfony, nor it looks even remotely like Symfony code.  So, what
> if, by some chance, you have been entrusted with maintenance of a
> huge legacy PHP codebase?

[more]

On the surface, it might look like a thankless, soul-sucking job,
piling hacks upon hacks, fixing breakage of seemingly unrelated
parts of application with each change, never being quite sure that
your code works as expected, and having little opportunity for
professional growth.  Most developers would be tempted daily
to throw out entire thing and start the Big Rewrite.

Yet I tend to agree with a well-known [Joel Spolsky post][joel-rewrite]
(14 years ago, mind you, I was 12 then) which states that
the Big Rewrite is usually the wrong thing to do:

* The Big Rewrite would take enormous developer resources (while
  you'll still need to keep maintaining the legacy codebase until the
  new one is complete).
* It is very hard to ensure that all use cases are covered correctly.
  The legacy code, with all its deficiencies, still works, and
  fulfills some business goals.  Most of these goals were likely
  never formalized anywhere.  During the Big Rewrite, it is trivial
  to miss some crucial detail, and base your new design on it in a way
  which would be hard to change.
* When (and if) the Big Rewrite is complete, users may resist
  migrating to the new version.  Whenever any missing feature,
  or a bug which wasn't there before, or even an UI change is found,
  it might be easier to revert than to wait for a fix.  This may
  be the case even if the app in question is an internal,
  line-of-business application; expect "Well it wasn't like
  *that* before!  Just keep the old version running for now"
  from your management.

Gradual refactoring of legacy code to Symfony or another modern
framework and modern code standards may seem nigh impossible.
However, the modular and flexible nature of Symfony makes it quite
feasible, if not entirely trivial in some aspects.

My current day job is the maintenance of a legacy PHP code base.
It is a line-of-business billing application for a sizable
web hosting company.  The application is a commercial
and closed-source product, having its code obfuscated with
[ionCube][ioncube].  Thankfully we have an unobfuscated earlier version of
this product for reference.  The application, which I'll
call **LegacyApp** (not its real name obviously), had been
extended in many ways before, with plugins, hooks, official APIs,
and at times some really evil monkey-patching hackery in
templates.  However there were limits to how much it was
possible to accomplish without refactoring or rewrite.

Since then, the legacy application was entirely wrapped in
new Symfony code.  It made it possible to bypass and/or rewrite
the worst parts of that app entirely.  All common web application
components (DI, session, database, events/hooks, UI translation,
authentication/authorization, tests) were fully and cleanly
integrated with legacy code.  So the LegacyApp is not
particularly legacy anymore, and new development for it
is a breeze compared to what it was like before.

So in this series of posts I'm going to elaborate on how
various components of Symfony can be integrated with legacy code.
I tried to abstract from the specifics of LegacyApp as much as
possible, so these recipes should work with some tweaks for
most legacy applications using same common patterns (and
more often, anti-patterns).  Have fun refactoring!

[php-fractal]: http://eev.ee/blog/2012/04/09/php-a-fractal-of-bad-design/
[symfony]: http://symfony.com/
[joel-rewrite]: http://www.joelonsoftware.com/articles/fog0000000069.html
[ioncube]: http://www.ioncube.com/