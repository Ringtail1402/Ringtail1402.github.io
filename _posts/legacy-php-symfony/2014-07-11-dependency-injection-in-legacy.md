---
layout: post-en
category : "en"
title: How to use Symfony Dependency Injection in legacy code
tags: [development, php, symfony2, legacy, cookbook]
---
{% include JB/setup %}

> Symfony Dependency Injection container is perhaps my favorite component of
> Symfony 2.

It is a non-trivial component to understand, though, with its multitudes of
YAML/XML files and weird classes in `DependencyInjection` namespaces and
bundle semantic configuration and whatnot.  In my first Symfony 2 project,
I was thoroughly confused by Dependency Injection (along with Doctrine's
data mapper semantics) and ended up treating classes in `DependencyInjection`
namespaces and YAML configs as magic incantations.  I began to realize what
DI was all about only after I wrote a sizable application based on
[Silex][silex] with its trivial Pimple container.  And for a large
application like our LegacyApp, DI is simply indispensable.

In my Symfony code, I define as much classes as I can as services.
It is not strictly necessary for controllers, console commands, and
Doctrine repositories (and dependency-injected constructors for
controllers may get quite daunting), but I find it very helpful to think
of services as basic building blocks of an application.  Apart from
a few Symfony-specific classes like `Bundle` subclasses, everything
is either a service, or created and managed by services.  And of course
dependencies on `ContainerInterface` are a no-no.  We want to use container
to wire up components of our application on startup, not to use it as
as "service locator" to retrieve random services whenever we feel like it.

Apart from consistency, a great reason to use Symfony DI is that it is
fairly simple and clean to call services from legacy code.  We have already
looked at some examples of calling legacy code from Symfony layer, but
in real life, the opposite is often necessary as well.  For example, we may have
just refactored some legacy `invoicegeneration.php` code with a bunch
of functions to a shiny new `InvoiceGenerator` class, registered via DI
as an `invoicing.generator` service.  We still want to keep `invoicegeneration.php`
and its functions with their APIs, though; they just need to be wrappers
for `InvoiceGenerator` methods now.  How can we do that?

> Turns out, as far as interfacing with legacy code goes, this is mostly trivial.

Look at your `AppKernel` class in `app/AppKernel.php`.  It inherits from
`Kernel` class of HttpKernel component.  Normally `AppKernel` is just a place
where you add your bundles in `AppKernel#registerBundles()` method.

As an aside, I found the differences between `HttpKernel`, `Kernel` and `AppKernel`
a point of some confusion.  Here is what each of these classes does:

* **`HttpKernel`** is a default implementation of `HttpKernelInterface`.  This is
  an extremely simple interface.  It literally has a single `handle()` method that accepts
  a HTTP `Request` and returns a HTTP `Response`.  `HttpKernel` turns
  a request into a response by issuing a few events via Symfony Event Dispatcher,
  and picking and calling a controller function via `ControllerResolverInterface`.
  Conceptually, `HttpKernel` is very similar to a Java servlet.

* **`Kernel`** is the central class of a Symfony application.  It wraps two main
  things any application has: the DI container and a bunch of bundles.  Upon
  its initialization (`Kernel#boot()`), it loads auto-generated container
  class (regenerating it if necessary), and calls `BundleInterface#boot()`
  function for all registered bundles.  `Kernel` also implements `HttpKernelInterface`,
  but its `handle()` method merely passes to `http_kernel` service, which is
  expected to be always registered.  The container is always available in `Kernel` via
  `getContainer()` method.  `Kernel` is abstract and must be subclassed by...

* **`AppKernel`** (default name) is a subclass of `Kernel`, which implements
  `registerBundles()` and `registerContainerConfiguration()` methods missing in
  `Kernel`.  The front controller script (`web/app.php` or `web/app_dev.php`
  by default) creates an `AppKernel` instance, a `Request` from PHP superglobals,
  and calls `AppKernel#handle()`.  `AppKernel` boots itself, initializing container,
  and executes request with `HttpKernel`, returning `Response`.  `Response`
  is then sent to client via `Response#send()` by front controller script.

Understanding this will help us make container available globally from legacy code.
Let us make `AppKernel` a singleton class:

{% highlight php %}
<?php

class AppKernel extends Kernel
{
    private static $instance;

    public function boot()
    {
        parent::boot();
        self::$instance = $this;
    }

    public static function getInstance()
    {
        return self::$instance;
    }

    public function registerBundles() { /* ... */ }

    public function registerContainerConfiguration(LoaderInterface $loader) { /* ... */ }
}
{% endhighlight %}

Singletons are bad; by now any junior developer knows that.  But so is legacy code, you know.
And in this case, singletons solve a problem in a useful way.  Of course no class other
than `AppKernel` should ever be a singleton in your app; that's what DI is for.

So by now you have probably realized what we are going to do in `invoicegeneration.php`:

{% highlight php %}
<?php

function generateInvoices()
{
    AppKernel::getInstance()->getContainer()->get('invoice.generator')->generateInvoices();
}
{% endhighlight %}

> Singleton and Service Locator!  Antipatterns in regular Symfony code, these are the
> best we can do in legacy layers of our application.  It's not as if we can tell
> container to inject a service into a top-level function, or maybe even top-level code.

As Symfony is always bootstrapped before any legacy code runs, this code is safe.
It's unwieldy though.  We may add a helper function or two at the end of `app/AppKernel.php`
to alleviate that:

{% highlight php %}
<?php

class AppKernel extends Kernel { /* ... */ }

function s($service)
{
    return AppKernel::getInstance()->getContainer()->get($service);
}

function p($parameter)
{
    return AppKernel::getInstance()->getContainer()->getParameter($parameter);
}
{% endhighlight %}

So now our wrapper function may be simplified to:

{% highlight php %}
<?php

function generateInvoices()
{
    s('invoice.generator')->generateInvoices();
}
{% endhighlight %}

If you're using an IDE like PhpStorm, and want to call more that a single method on
your service, it might help to add a phpDoc line to get autocompletion:

{% highlight php %}
<?php

/* don't forget to add "use" line for your class */

function generateInvoices()
{
    /** @var InvoiceGenerator $invoice_generator */
    $invoice_generator = s('invoice.generator');
    $invoice_generator->generateInvoices();
    $invoice_generator->doSomeOtherStuff();
    /* ... */
}
{% endhighlight %}

Even better.  Now, assuming you use PhpStorm, you may eventually do a
*Find Usages...* on `generateInvoices()` and refactor its calls to use
`InvoiceGenerator` directly.  But for now, our `s()` and `p()` wrapper
functions are going to be very helpful for our glue code.

[silex]: http://silex.sensiolabs.org/