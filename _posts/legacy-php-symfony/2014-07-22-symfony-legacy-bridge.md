---
layout: post-en
category : "en"
title: "Symfony and Legacy Code: Calling from Symfony into legacy code"
tags: [php, symfony2, symfony-legacy]
---
{% include JB/setup %}

> Having stated our premise in [previous post]({% post_url legacy-php-symfony/2014-07-21-introduction %}),
> let's have a look at actually setting up a Symfony project
> capable of interoperating with LegacyApp.

Our big goal is to make LegacyApp a *part* of larger Symfony-based application.
It should be wrapped entirely in Symfony code, and its URL and CLI scripts launched only
from Symfony front controllers.

Why?  This is, in practical terms, the only viable choice there is.  We don't want Symfony and
legacy apps to be entirely separate and working side by side.  Most of application features
depend on each other in various ways, and if the only way new and old code can interact
is through database, this is going to be tricky.

Another theoretical option is to keep LegacyApp a first-class citizen and make it
call into Symfony code for new code paths.  This is very inconvenient as:

* We lose benefits of having a single front controller in application
* We lose ability to do useful things in global event listeners like `KernelEvents::REQUEST`
* We lose ability to have our routing managed by Symfony
* We probably will not use Symfony-based controllers at all, keeping "one controller =
  one .php script" approach
* Depending on how our LegacyApp bootstrap code works, it might be difficult to make
  sure Symfony is initialized at a given point of our code at all

And how are we going to wrap a PHP codebase, with possible hundreds of thousands of lines of
code large, in a Symfony app?  Well, just define a single fallback controller,
handling any request ending in ".php", `require` a legacy script in that controller,
getting its output with `ob_start()`/`ob_get_clean()`, and push that output into a
`Response` object!  It really is as simple as that.  The basic idea is described in
[this][theodo-slideshare] presentation.

When you take into account all use cases
and all quirks of legacy app, it becomes a little more complicated though.  So in this
post we'll only examine how to create a universal "bridge" service for calls from Symfony
into legacy code.  The actual fallback controller for legacy URLs is a subject for the
next post.

We will start by creating a Symfony project; I'll assume you can do that without much
difficulty.  Then we'll put our legacy code in a subdirectory of this project:

<pre><code>*
+- /app
|  +- /config
+- /vendor
|- +- ...
+- /src
|- +- ...
+- /web
+- <b>/legacy-app</b>  &lt;-- put it here
</code></pre>

Only `/web` directory, containing front controller scripts `app.php` and possible `app_dev.php`,
and static content, should be ever accessible from the web.  We must put our legacy code base
*outside* `/web` dir.

Just as Symfony app has a single entry point in its `web/app.php` script,
we will make a single entry point for LegacyApp in a `LegacyBridge` class.
We'll put it into `Enotogorsk\LegacyApp\LegacyInteropBundle` bundle for the
sake of this example.

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle;

class LegacyBridge
{
    private $legacy_app_path;

    public function __construct($legacy_app_path)
    {
        $this->legacy_app_path = $legacy_app_path;
    }

    public function enterLegacyApp($filename)
    {
        require $this->legacy_app_path . '/' . $filename;
    }
}
{% endhighlight %}

This method, consisting of a single `require`, and accepting a legacy PHP
script to be executed/evaluated as an argument, is going to be our starting point.
Depending on our LegacyApp, we may want to add some setup to this method:

* There is likely a single "bootstrap" .php file, included by any other script
  of LegacyApp.  We may as well make it a default argument:

{% highlight php %}
<?php

class LegacyBridge
{
    public function enterLegacyApp($filename = 'init.php')
    {
        /* ... */
    }
}
{% endhighlight %}

* This way, calling `enterLegacyApp()` without parameters will just bootstrap legacy
  part of our app, but not do anything else.  Which is what we'll often want.

* On the other hand, legacy code probably is not going to handle executing two or more
  top-level .php files in one request well.  I.e. there may be `/cart.php` and `/invoice.php`
  files in the LegacyApp, both mapping to an URL; and we probably cannot include both
  files at once.  So let's guard against double call of `enterLegacyApp()` for anything
  but default bootstrap .php file:

{% highlight php %}
<?php

class LegacyBridge
{
    public function enterLegacyApp($filename = 'init.php')
    {
        // defined('IN_LEGACYAPP') is an example; check for anything at all
        // (function, class, constant) that is defined in init.php
        if (defined('IN_LEGACYAPP')) {
            if ($filename == 'init.php') {
                // do nothing if asked to include bootstrap part twice
                return;
            }
            throw new \RuntimeException("LegacyApp already initialized when trying to include '$filename'.");
        }

        /* ... */
    }
}
{% endhighlight %}

* Legacy code *loves* global variables.  If the code that was top-level in LegacyApp
  defined any variables, they would automatically become global, and code in functions/classes
  might try to access them with `global` keyword.  On the other hand, with our wrapper
  method, former top-level code would be included at function level, and its variables
  won't become global.  So we'll need to explicitly define *any* possible global
  variables with `global` keyword in `enterLegacyApp()`:

{% highlight php %}
<?php

class LegacyBridge
{
    public function enterLegacyApp($filename = 'init.php')
    {
        /* ... */

        global $_I18N;
        global $license;
        global $template;
        global $cron;
        // and so on

        /* ... */
    }
}
{% endhighlight %}

* In our actual app, about 30 global variables are declared this way.  It's okay
  if some or all of these variables will never be assigned/accessed during a
  particular request; we just need to make sure they're there when they are needed.

* Legacy code likes setting `error_reporting` to something very permissive, up to
  disabling error reporting at all.  Sadly we'll need to accomodate it, unless
  we want our app to crash with `E_WARNING` or `E_NOTICE` on every second request:

{% highlight php %}
<?php

class LegacyBridge
{
    public function enterLegacyApp($filename = 'init.php')
    {
        /* ... */

        // report only errors (you may need to go as far as error_reporting(0))
        $old_error_reporting = error_reporting(E_ERROR);

        require $this->legacy_app_path . '/' . $filename;

        // restore default error reporting level for new code
        error_reporting($old_error_reporting);
    }
}
{% endhighlight %}

* Legacy code may make assumptions about current dir, usually that it equals
  directory of the .php script being requested.  Accomodate that with:

{% highlight php %}
<?php

class LegacyBridge
{
    public function enterLegacyApp($filename = 'init.php')
    {
        /* ... */

        // you may want to remember current dir here and restore it later,
        // if your own code is also sensitive to current dir
        chdir(dirname($this->legacy_app_path . '/' . $filename));

        /* ... */
    }
}
{% endhighlight %}

These aren't all issues you need to deal with, but we'll examine HTTP- and
CLI-specific ones in the following posts.  Don't forget to register `LegacyBridge`
class as a service in your dependency injection configuration.  In our example,
we made `$legacy_app_path` a constructor argument, so that it may be set with
a config file.

We cannot wrap legacy URLs with new code yet, but we can already call legacy
code from new Symfony-based code!  Just define wrapper functions in `LegacyBridge`
or another class that depends on it, like that:

{% highlight php %}
<?php

class LegacyBridge
{
    /* Assume we have update_invoice($invoiceid) function defined in
       includes/invoicefunctions.php file in LegacyApp */
    public function updateInvoice(Invoice $invoice)
    {
        // make sure legacy code is bootstrapped
        $this->enterLegacyApp();

        // include file with the function required.  We want to use plain require
        // rather than enterLegacyApp() here, as invoicefunctions.php is just
        // a collection of functions, rather that top-level executable code
        if (!function_exists('update_invoice')) {
            require_once $this->legacy_app_path . '/includes/invoicefunctions.php';
        }

        // call into legacy code.  As an example, we also convert an argument here;
        // the LegacyBridge method accepts an Invoice object, presumably a Doctrine
        // entity, while legacy function accepts just invoice ID.  (If this is really
        // a Doctrine entity, you'll probably want to call EntityManager#refresh()
        // on it to pick any changes persisted from legacy code)
        \update_invoice($invoice->getId());
    }
}
{% endhighlight %}

As a result, this `LegacyBridge` class would become a facade to all of legacy code.
You may split it in several classes if it gets too large; the important part is that
legacy APIs are well isolated in these particular classes.  This way, any uses
of these APIs may be easily detected, and eventually refactored to new code.

Still, we don't have much of that new code at the moment.  For most or all of its
URLs we just want legacy pages to be shown as is at the moment.  In the
[next post]({% post_url legacy-php-symfony/2014-07-23-wrap-legacy-url %})
we'll try to do exactly that.

[theodo-slideshare]: http://www.slideshare.net/fabrice.bernhard/modernisation-of-legacy-php-applications-using-symfony2-php-northeast-conference-2013
[symfony-installation]: http://symfony.com/doc/current/book/installation.html
