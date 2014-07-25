---
layout: post-en
category : "en"
title: "Symfony and Legacy Code: Interface with legacy $_SESSION"
tags: [php, symfony2, symfony-legacy]
---
{% include JB/setup %}

> Our LegacyApp uses built-in PHP `$_SESSION` facility to store a number
> of values for user: ID of currently logged-in client and/or admin, hashes
> of their passwords, shopping cart, value for anti-CSRF token generation,
> etc.  We would certainly want to have access to these values from Symfony code.

Symfony 2, on the other hand, has `Session` class, which is a part of
its HttpFoundation component.  Symfony session implementation abstracts
out session storage, and makes session a part of a `Request` object.
It has built-it support for flash attributes (auto-deleting after access)
and other goodies.

Making `$_SESSION` and `Session` interoperable requires us to:

1. Make sure `$_SESSION` and `Session` work with same session which
   is initialized and started in a uniform way;

2. Make sure `$_SESSION` and `Session` APIs are able to access same
   values in session.

The first part requires us to decide whether we want session to be started
by Symfony or by legacy code.  Legacy part of our app probably sets up its
session with `session_*()` calls.  At the very least it likely calls
`session_name()` to set session cookie name, and then `session_start()`.

Symfony has an option to just pick up session already started by other code.
Just use built-in [`PhpBridgeSessionStorage`][php-bridge-session-storage]
class for session storage.  It can be set up in `app/config/config.yml` like this:

{% highlight yaml %}
framework:
    session:
        storage_id: session.storage.php_bridge
{% endhighlight %}

The downside, of course, is that you now have to make sure to somehow bootstrap your
legacy app on every request, so that it would start session before any Symfony code
tries to touch it.

For our LegacyApp, we chose another approach: make Symfony start session.
In this case, we may override `Session` class or define a configurator class
for `session` service, which would do initialization right after `Session`
object is first instantiated.  The second approach would look like this:

{% highlight yaml %}
services:
    # Override session service
    session:
        class: "%session.class%"
        arguments: [ "@session.storage", "@session.attribute_bag", "@session.flash_bag" ]
        configurator: [ "@session.legacy_configuration", initLegacySession ]

    # Define configurator service
    session.legacy:
        class: Enotogorsk\LegacyApp\LegacyInteropBundle\Session\SessionConfigurator
{% endhighlight %}

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Session;

class SessionConfigurator
{
    public function initLegacySession(SesssionInterface $session)
    {
        if (php_sapi_name() == 'cli') {
            return;  // sanity check
        }

        $session_name = 'LegacyApp';  // or some logic here
        $session->setName($session_name);
    }
}
{% endhighlight %}

Putting `$session->start()` in this class is a bad idea though;
we don't want session to auto-start that early.  We may just make sure
session is started in our main legacy bridge class:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle;

class LegacyBridge
{
    private $session;

    public function __construct(SessionInterface $session /*, other args */)
    {
        $this->session = $session;
        /* ... */
    }

    public function enterLegacyApp($filename, $skip_buffering = false)
    {
        if (php_sapi_name() != 'cli') {
            $this->session->start();  // this is safe to do even if session is already started
        }

        /* ... */
    }
}
{% endhighlight %}

This way, session will always be pre-started for any legacy code calls.
You may want to remove `session_start()` and other session calls from legacy code
if possible.  `session_start()` in particular will issue a `E_NOTICE`-level error
if session is already started.

> So we have dealt with the first part; the second problem still remains.

It might not be immediately obvious, but `Session` service in Symfony cannot give
you access to a value like `$_SESSION['userid']` or `$_SESSION['cart']` without
additional code and configuration.  Simple `$session->get('userid')` will not
do what you expect.

Rather than managing stored values directly, Symfony session consists of a number
of **bags**, which must implement only a very minimal `SessionBagInterface`.
By default, session has an attribute bag (which is what `Session#get()` and
`Session#set()` work with), a flash bag, and a metadata bag.  You can add
more bags if you want.  A bag is free to implement any interface and semantics
it likes.  E.g. `AttributeBag` is little more than an array wrapper, while
`FlashBag` also wraps an array but removes its elements on `get()`.

Physically (assuming you use default `NativeSessionStorage` implementation),
each bag is a value in `$_SESSION` superglobal.  `SessionBagInterface#getStorageKey()`
must return key in `$_SESSION` under which the bag contents will be stored.
Upon session start, `SessionBagInterface#initialize()` will be called with
a *reference* to a value in `$_SESSION`.  This way, any modifications on
bag values will be stored automatically, and bags may be simple self-contained
classes unaware of session storage.

By default, `$_SESSION['_sf2_attributes']` is an attribute bag, `$_SESSION['_sf2_flash']`
is a flash bag, and `$_SESSION['_sf2_meta']` is a metadata bag.  So `$session->get('userid')`
will give you `$_SESSION['_sf2_attributes']['userid']` value, while our LegacyApp
uses `$_SESSION['userid']`, blissfully unaware of bags and other Symfony stuff.

In principle, we can of course just use Symfony session support for new code only,
and just access raw `$_SESSION` for values set by legacy code whenever we need to.
This might not be such a terrible idea. `$_SESSION` is a workable session
abstraction as it is, and it can be mocked for testing, etc.  Still, it would be
nice to get our userid and whatnot in `Session`, for consistency if nothing else.

> What we want, then, is to add more bags to session, one for every legacy value
> we want to access.

This is simple enough for things like a shopping cart, which are likely
stored in `$_SESSION` as a subarray.  We can use more `AttributeBag`s for that,
or, more likely, its `NamespacedAttributeBag` subclass, which stores a
multidimensional array.  `$_SESSION['userid']` however is not an array; it's
just an integer value.  So it appears we need a bag class which stores
a simple scalar value.  That class would be fairly trivial, but Symfony does not
offer such a class by default.

We can do this by hand, or use a readymade solution, such as
**TheodoEvolutionSessionBundle**.  This bundle, available via Composer,
offers a `ScalarBag` class, which does exactly what we described in the previous
paragraph, and a `BagManager` class, which auto-adds bags for specified legacy
session values.  The bundle also has support for interfacing with symfony 1.x
sessions, so you may look into that if your legacy app is in fact a
symfony 1.x product.

Theodo is a French company which apparently makes a living porting
legacy PHP code to Symfony.  It is in fact their [presentation][theodo-slideshare]
that made me realize that wrapping entire legacy app in Symfony is actually
a feasible approach.  TheodoEvolutionSesssionBundle is available on
[GitHub][theodo-github].

Put this in your `composer.json` and do `composer update` to install
TheodoEvolutionSessionBundle:

{% highlight javascript %}
{
    /* ... */
    "require": {
        /* ... */
        "theodo-evolution/session-bundle": "~1.0"
    }
}
{% endhighlight %}

Then enable new bundle in `app/AppKernel.php`.

You will need to create a short configuration class for TheodoEvolutionSessionBundle,
implementing `BagManagerConfigurationInterface`.  Here is what it should look like:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Session;

use Theodo\Evolution\Bundle\SessionBundle\Manager\BagManagerConfigurationInterface;

class LegacyBagManagerConfiguration implements BagManagerConfigurationInterface
{
    public function getNamespaces()
    {
        /* This is the important part.  Just return an array of all $_SESSION keys
           which you want to be available as bags. */
        return ['userid', 'userpw', 'adminid', 'adminpw', 'language', 'template', 'cart'];
    }

    public function getNamespace($key)
    {
        return $key;
    }

    public function isArray($namespaceName)
    {
        /* If some of the $_SESSION values are in fact arrays, you should return true
           here for corresponding keys. */
        if ($namespaceName == 'cart') {
            return true;
        }

        return false;
    }
}
{% endhighlight %}

Then, tell TheodoEvolutionSessionBundle to use default bag manager with your
configuration class.  Add this to `app/config/config.yml`:

{% highlight yaml %}
theodo_evolution_session:
    bag_manager:
        class: Theodo\Evolution\Bundle\SessionBundle\Manager\BagManager
        # Replace with your class name of course
        configuration_class: Enotogorsk\LegacyApp\LegacyInteropBundle\Session\LegacyBagManagerConfiguration
{% endhighlight %}

That's all.  TheodoEvolutionSessionBundle registers a `KernelEvents::REQUEST` listener,
which call `initialize()` for bag manager you chose.  Default `BagManager` simply adds
bags to session.  These will be `ScalarBag`s for simple keys, and `NamespacedAttributeBag`s
for arrays.  `ScalarBag` is a bag class storing a single scalar value, just as we
have discussed earlier.

So how would you access `userid` value from session now?

{% highlight php %}
$userid = $session->getBag('userid')->get();
$session->getBag('userid')->set($userid);
{% endhighlight %}

You should probably define some constants for `userid` etc.  Or even subclass
`Session` and define some custom getters and setters:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Session;

use Symfony\Component\HttpFoundation\Session\Session;

class LegacyAwareSession extends Session
{
    public function setUserId($userid)
    {
        $this->getBag('userid')->set($userid);
    }

    public function getUserId()
    {
        return $this->getBag('userid')->get();
    }

    /* and so on */
}
{% endhighlight %}

Override `session` service definition in dependency injection configs
to make Symfony use your session class.

That's all; now you have legacy and Symfony code playing along nicely
when it comes to session.  We used `userid` value, presumably set by
legacy login code, as an example throughout this post.  You should not
however work with `userid` value directly!  A much better approach
would be bridging old login code with Symfony Security subsystem,
which is perfectly possible even if all you have is that simple little
`userid` value.  We will examine this technique in a later post.
For now, just remember that you should use session for cart data and
other things unrelated to auth.

[php-bridge-session-storage]: http://symfony.com/doc/current/components/http_foundation/session_php_bridge.html
[theodo-slideshare]: http://www.slideshare.net/fabrice.bernhard/modernisation-of-legacy-php-applications-using-symfony2-php-northeast-conference-2013
[theodo-github]: https://github.com/theodo/TheodoEvolutionSessionBundle
