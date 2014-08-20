---
layout: post-en
category : "en"
title: "Symfony and Legacy Code: EventDispatcher and legacy hooks"
tags: [php, symfony2, symfony-legacy]
---
{% include JB/setup %}

The Symfony [EventDispatcher][event-dispatcher-doc] component is among
the simplest yet extensively used both by Symfony itself and by application code.
Its function is very simple: there is one central `EventDispatcherInterface`
service, which can dispatch arbitrary events (defined by name and optionally by an
instance of `Event`-derived class), and register listeners for these events.
EventDispatcher is invaluble in writing extensible, loosely coupled code.

A good example of EventDispatcher use is Symfony's central `HttpKernel` class,
which, as we saw [before]({% post_url legacy-php-symfony/2014-07-25-dependency-injection-in-legacy %}),
transforms a HTTP `Request` into a `Response`.  It does that by resolving and executing
a controller function.  Controllers are typically mapped to URLs via Routing component;
however `HttpKernel` is entirely unaware of Routing classes and their implementation.
Instead there is a `RouterListener` class which glues Routing to `HttpKernel` by locating
appropriate controller name, and setting `_controller` attribute on `Request` (or throwing a
`NotFoundHttpException` if no routes match the request).  `HttpKernel` may then resolve
controller name to an actual controller function and its arguments using default
`ControllerResolverInterface` implementation, and call this function to render a `Response`.

> That's all fine and good, but sizable legacy applications attempting to be
> somewhat extensible typically implement some kind of events/hooks/whatever-they-call-it
> system themselves.  So can we route the legacy hook APIs through EventDispatcher?
> Why of course we can.

We will assume here that our LegacyApp wants us to put all our custom hooks in its `includes/hooks` directory.
All .php files there get autoloaded on startup, and are expected to look like this:

{% highlight php %}
<?php

register_hook('InvoicePaid', 10, 'my_invoice_paid_hook');

function my_invoice_paid_hook(array $params = array())
{
    $invoice_id = $params['id'];
    /* do our custom processing here */
}
{% endhighlight %}

Then the hooks may be called like that:

{% highlight php %}
<?php

/* ... some invoice payment handling code ... */
run_hook('InvoicePaid', array('id' => $invoice_id));
{% endhighlight %}

This is it.  Simple and to the point.  Not the worst API ever, actually.

Ideally we want to have full bidirectional compatibility between this system and
Symfony EventDispatcher:

* `run_hook()` called from legacy code should be able to hook into Symfony
  event listeners/subscribers;
* Dispatching an event via Symfony EventDispatcher should be able to run
  legacy hook functions added with `register_hook()`.

Let's start with an event class which will be compatible with legacy hook
parameters.

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Event;

use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\EventDispatcher\Event;

class LegacyAwareEvent extends Event
{
    /**
     * @var boolean  is this event raised by run_hook() rather than EventDispatcher?
     */
    protected $from_legacy = false;

    /**
     * @return boolean
     */
    public function isFromLegacy()
    {
        return $this->from_legacy;
    }

    /**
     * Constructs an event object from a legacy hook parameter array.
     *
     * @param ContainerInterface $container
     * @param string             $event_name
     * @param array              $params
     * @return LegacyAwareEvent
     */
    static public function fromHookArgs(ContainerInterface $container, $event_name, array $params = [])
    {
        $e = new static();
        $e->from_legacy = true;
        return $e;
    }

    /**
     * Converts an event object to a legacy parameter array.
     *
     * @return array
     */
    public function toHookArgs()
    {
        return [];
    }
}
{% endhighlight %}

As you see, this event class can serialize itself into a plain parameter array,
and reconstruct an instance of itself from the same array.  The latter operation
may require additional services, so we pass the DI container here.

This base class does not actually have any extra parameters, so we probably want to
define some subclasses looking like this:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Event;

class InvoiceEvent extends LegacyAwareEvent
{
    // this hook has a single 'id' parameter, which is the ID of invoice being paid
    const INVOICE_PAID = 'InvoicePaid';

    private $invoice;

    public function __construct(Invoice $invoice)
    {
        $this->invoice = $invoice;
    }

    public function getInvoice()
    {
        return $this->invoice;
    }

    static public function fromHookArgs(ContainerInterface $container, $event_name, array $params = [])
    {
        // get invoice repository service from DI container to reconstruct an Invoice object from its ID
        $invoice_repo = $container->get('invoice.repository');
        switch ($event_name) {
            case static::INVOICE_PAID:
                $e = new static($invoice_repo->find($params['id']));
                $e->from_legacy = true;
                return $e;

            default:
                throw new \RuntimeException("Unknown hook $event_name.");
        }
    }

    public function toHookArgs()
    {
        return ['id' => $this->invoice->getId()];
    }
}
{% endhighlight %}

> Thus we have an event class.  We may raise this event with `EventDispatcher` class,
> register event listeners/subscribers to listen to this event.  But we're still missing
> code which will actually convert events to hook calls and back, calling `fromHookArgs()`
> and `toHookArgs()` whenever necessary.

The basic `EventDispatcher` class, or rather `ContainerAwareEventDispatcher`, is registered
in Symfony's DI container as `event_dispatcher` service.  You can override `event_dispatcher`
definition, substituting a subclass with required new functionality.  In fact this is what
Symfony does by default in debug mode, using `TraceableEventDispatcher` to log events.

So we want to write a subclass of `EventDispatcher` which will be able to handle
both Symfony-level events and legacy hooks.  Let's start with Symfony --> legacy calls.

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Event;

use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\EventDispatcher\ContainerAwareEventDispatcher;
use Symfony\Component\EventDispatcher\Event;

class LegacyAwareEventDispatcher extends ContainerAwareEventDispatcher
{
    private $legacy;

    public function __construct(ContainerInterface $container, LegacyBridge $legacy)
    {
        parent::__construct($container);
        $this->legacy = $legacy;
    }

    /**
     * Handles an event.
     * This method is called by public EventDispatcher#dispatch() method,
     * and does the actual work.
     *
     * @param array  $listeners
     * @param string $eventName
     * @param Event  $event
     */
    protected function doDispatch($listeners, $eventName, Event $event)
    {
        // default listeners
        parent::doDispatch($listeners, $eventName, $event);

        // extra handling for legacy hooks
        if ($event instanceof LegacyAwareEvent) {
            // prevent recursion
            if ($event->isFromLegacy()) {
                return;
            }

            // build arguments array and run legacy hook
            // add an extra flag signalling that the hook call originated from new code
            $args = $event->toHookArgs();
            $args['from_symfony'] = true;
            $this->legacy->runHook($eventName, $args);
        }
    }
}
{% endhighlight %}

We encapsulate the actual `run_hook()` legacy function in `LegacyBridge#runHook()` method;
see the [legacy bridge]({% post_url legacy-php-symfony/2014-07-22-symfony-legacy-bridge %}) chapter.
You can also see that any legacy hooks will be executed *after* Symfony-level event listeners.

This is straightforward, but there is one subtlety; the default `EventDispatcher#dispatch()`
implementation actually will not call `EventDispatcher#doDispatch()` at all if no listeners
are registered for the event.  Since our `EventDispatcher` implementation does not really register
legacy hook functions as listeners, this condition must be omitted, and the rest of `dispatch()`
method is copypasted from parent classes:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Event;

class LegacyAwareEventDispatcher extends ContainerAwareEventDispatcher
{
    /* ... */

    public function dispatch($eventName, Event $event = null)
    {
        $this->lazyLoad($eventName);

        if (null === $event) {
            $event = new Event();
        }

        $event->setDispatcher($this);
        $event->setName($eventName);
        $this->doDispatch($this->getListeners($eventName), $eventName, $event);
        return $event;
    }
}
{% endhighlight %}

Now for the reverse direction: make it possible for hooks called from legacy code to be
handled by Symfony event listeners/subscribers.  The logical thing is to add some
auto-generated stub hook functions, which will call `EventDispatcher#dispatch()`.
We'll probably need to keep some sort of registry of possible legacy hook names and
corresponding event classes.

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Event;

class LegacyAwareEventDispatcher extends ContainerAwareEventDispatcher
{
    /* ... */

    /**
     * @var array   hook name => full event class name
     */
    private $hook_mapping;

    public function __construct(ContainerInterface $container, LegacyBridge $legacy, array $hook_mapping)
    {
        parent::__construct($container);
        $this->legacy = $legacy;
        $this->hook_mapping = $hook_mapping;
    }

    public function registerLegacyHooks()
    {
        foreach ($this->hook_mapping as $hook_name => $event_class) {
            // add a hook function for all known events
            // priority of -1000 will presumably ensure that event listeners
            // will be called before hook functions
            $boilerplate = <<<EOF
function __eventdispatcher_$hook_name(\$params)
{
    // prevent recursion
    if (!empty(\$params['from_symfony'])) {
        return;
    }

    // construct and handle event
    \$event = f('event_dispatcher')->createLegacyEvent('$hook_name', \$params);
    f('event_dispatcher')->dispatch('$hook_name', \$event);
}
EOF;
            eval($boilerplate);
            $this->legacy->registerHook($hook_name, -1000, "__eventdispatcher_$hook_name");
        }
    }

    public function createLegacyEvent($hook_name, array $params = [])
    {
        $class = $this->hook_mapping[$hook_name];
        return $class::fromHookArgs($this->getContainer(), $hook_name, $params);
    }
}
{% endhighlight %}

We use `eval()` here as a quick and simple solution.  If you can use an anonymous function here,
by all means do; in our case legacy `register_hook()` didn't handle it properly.  The global
`f()` function, used to access services in DI container from legacy scope, was examined
[before]({% post_url legacy-php-symfony/2014-07-25-dependency-injection-in-legacy %}).

Note that we use two different flags, `$params['from_symfony']` and `LegacyAwareEvent#$from_legacy`,
to remember the origin of event.  Both of them are required to prevent recursion (legacy -> Symfony
-> legacy -> ...).

Now, binding it all together.  `LegacyAwareEventDispatcher#registerLegacyHooks()` needs to be
called from a stub in legacy `includes/hooks` dir, named, say, `__eventdispatcher.php`:

{% highlight php %}
<?php

f('event_dispatcher')->registerLegacyHooks();
{% endhighlight %}

Finally, override `event_dispatcher` service definition.  It will look like this (assuming
YAML configuration):

{% highlight yaml %}
parameters:
  event_dispatcher.class: Enotogorsk\LegacyApp\LegacyInteropBundle\Event\LegacyAwareEventDispatcher

  # Hooks to event classes mapping is defined here.  Hooks not listed in this parameter
  # will not be mapped to events.
  event_dispatcher.hook_mapping:
    InvoicePaid: Enotogorsk\LegacyApp\LegacyInteropBundle\Event\InvoiceEvent
    # ...

services:
  event_dispatcher:
    class: "%event_dispatcher.class%"
    arguments: [ "@service_container", "@legacy", "%event_dispatcher.hook_mapping%" ]
{% endhighlight %}

Symfony will still wrap our custom `event_dispatcher` service in `TraceableEventDispatcher`
class for logging in debug mode, which is nice.  Now, hopefully, you'll be able to handle
legacy hooks with modern code.

Of course these code examples (as in other posts) are (loosely) based on our specific
LegacyApp and its specific hook API.  That's what they are, examples.  Your legacy app
will have its own API and you'll have to design your interface with it by yourself,
but hopefully this post will still be of some use.

[event-dispatcher-doc]: http://symfony.com/doc/current/components/event_dispatcher/introduction.html