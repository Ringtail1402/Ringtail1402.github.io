---
layout: post-en
category : "en"
title: "Symfony and Legacy Code: Wrapping legacy URLs"
tags: [php, symfony2, symfony-legacy]
---
{% include JB/setup %}

> We [can now]({% post_url legacy-php-symfony/2014-07-22-symfony-legacy-bridge %}) call
> into legacy code from new Symfony-based code with ease.  That will come useful
> for replacing old LegacyApp pages or creating entirely new ones.  But what about pages
> which we don't want to touch for now?

For that we should write a `LegacyController` class, which will handle all requests like
`GET /path/to/legacy/script.php`, and render output of that script; that approach was
briefly mentioned in the previous post.  In this way, URLs won't change, and legacy links won't break.

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Controller;

use Enotogorsk\LegacyApp\LegacyInteropBundle\LegacyBridge;
use Sensio\Bundle\FrameworkExtraBundle\Configuration as MVC;

/**
 * @MVC\Route(service="legacy.controller")
 */
class LegacyController
{
    private $legacy;

    public function __construct(LegacyBridge $legacy)
    {
        $this->legacy = $legacy;
    }

    /**
     * @MVC\Route("/{filename}.php", name="_legacy", requirements={"filename": ".+"})
     */
    public function legacyAction($filename)
    {
        // ???
    }
}
{% endhighlight %}

We're using `Route` annotation from `FrameworkExtraBundle` for simplicity.
I highly recommend using `FrameworkExtraBundle` annotations;
unlike say Doctrine entities there is no real downside to using annotations
in controllers.  As usual, we're omitting dependency injection configuration
(this controller is defined as a service).

The `requirements={"filename": ".+"}` part is important, as it makes sure
`filename` parameter will catch even URLs with embedded slashes, like
`/a/b/c.php`.  (I vaguely remember doing this was super difficult in symfony 1.
You had to subclass router or something.)

We put `.php` in route pattern explicitly so that other routes not ending
with `.php` may be captured by other Symfony controllers.  However this means that
our route won't catch e.g. `/admin/` which might be a shortcut for `/admin/index.php`.  These
cases have to defined explicitly:

{% highlight php %}
<?php

/**
 * @MVC\Route("/{filename}.php", name="_legacy", requirements={"filename": ".+"})
 * @MVC\Route("/", name="_legacy_index", defaults={"filename": "index"})
 * @MVC\Route("/admin/", name="_legacy_admin_index", defaults={"filename": "admin/index"})
 */
{% endhighlight %}

Now, we can execute legacy script requested simply by calling
`LegacyBridge#enterLegacyApp($filename . '.php')`.  But first we want to capture
output of that script with output buffering, a rather neat feature of PHP:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle;

class LegacyBridge
{
    /* ... */

    public function enterLegacyApp($filename)
    {
        /* skip setup, see previous post */

        ob_start();
        require $path;
        $response = ob_get_clean();

        return $response;
    }
}
{% endhighlight %}

That will capture response body nicely.  We only need to convert it to `Response`
object in controller:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Controller;

/* "use" declarations and annotations skipped */

class LegacyController
{
    /* ... */

    public function legacyAction($filename)
    {
        $response = $this->legacy->enterLegacyApp($filename . '.php');
        $response = new Response($response);
        return $response;
    }
}
{% endhighlight %}

As usual, there are some complications.  `new Response($string)` will
create a simple response with status *200 OK* and content type `text/html`.
This is what we usually want, but what if legacy code attempted to
render JSON or to do a redirect or to return *403 Forbidden* or whatever.
All of these things must be handled manually.

Legacy code will use built-in `header()` function to add response
headers, including `Location:` for redirects and status lines starting
with `HTTP/1.x` for setting status code.  Output buffering normally
does not affect headers, but sending a `Response` object always sets
status code and `Content-Type` header, which override these headers
if they were set up by legacy code earlier.  So we need to do something like:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Controller;

/* "use" declarations and annotations skipped */

class LegacyController
{
    /* ... */

    public function legacyAction($filename)
    {
        $response = $this->legacy->enterLegacyApp($filename . '.php');
        $response = new Response($response);

        // preserve status code
        $response->setStatusCode(http_response_code());

        // examine headers
        foreach (headers_list() as $header) {
            // preserve Content-Type:
            if (stripos($header, 'content-type: ') === 0) {
                preg_match('/^content-type: (.*)$/i', $header, $matches);
                $response->headers->set('Content-Type', $matches[1]);
            }

            // handle Location: by creating a RedirectResponse
            if (stripos($header, 'location: ') === 0) {
                preg_match('/^location: (.*)$/i', $header, $matches);
                $response = new RedirectResponse($matches[1], http_response_code());
            }
        }

        return $response;
    }
}
{% endhighlight %}

Anything else?  Well, if legacy code examines `$_SERVER['PHP_SELF']` or
`$_SERVER['SCRIPT_NAME']` values, it will see the Symfony front controller script
there instead of itself.  If this is a concern, we may just override these values.
The "real" URL for legacy script called through this legacy controller is
something like `/app.php/path/to/legacy/script.php`.  This URL is available
in `$_SERVER['DOCUMENT_URI']` even if `/app.php` part is hidden by
URL rewriting (as it should be).  So we may just get `$_SERVER['DOCUMENT_URI']`
and cut out the `/app.php` or `/app_dev.php` part:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Controller;

/* "use" declarations and annotations skipped */

class LegacyController
{
    /* ... */

    public function legacyAction($filename)
    {
        // attempt to hide Symfony front controller
        if (isset($_SERVER['DOCUMENT_URI'])) {
            $fake_script_name = preg_replace('#^/[^.]+\.php#', '', $_SERVER['DOCUMENT_URI']);
            $_SERVER['PHP_SELF'] = $fake_script_name;
            $_SERVER['SCRIPT_NAME'] = $fake_script_name;
        }

        /* rest of action skipped */
    }
}
{% endhighlight %}

The last complication we will examine is legacy code relying on
`register_globals` setting.  As we all know, `register_globals` is a
terrible terrible idea which should not have been used by anybody.
Naturally legacy code absolutely loves it.

`register_globals` emulation is trivial; the only issue is that
it has to be done from `LegacyBridge#enterLegacyApp()` function, as it
is the spot where `require` on legacy code is called:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Controller;

/* "use" declarations and annotations skipped */

class LegacyController
{
    /* ... */

    public function legacyAction(Request $request, $filename)
    {
        /* setup skipped */

        $response = $this->legacy->enterLegacyApp($filename . '.php', $request);

        /* rest of action skipped */
    }
}
{% endhighlight %}

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle;

class LegacyBridge
{
    /* ... */

    public function enterLegacyApp($filename, Request $request = null)
    {
        // extract $_POST and $_GET variables to emulate register_globals
        if ($request) {
            if ($request->request instanceof ParameterBag) {
                extract($request->request->all());
            }
            if ($request->query instanceof ParameterBag) {
                extract($request->query->all());
            }
        }

        /* rest of function skipped */
    }
}
{% endhighlight %}

As a final touch, we should add a check that the requested `$filename`
actually exists, and throw `NotFoundHttpException` for missing filenames,
so that the user will see a 404 error page instead of a scary 500 one.
This is left as an exercise for the reader.

Whew.  That was fun.  Now all of our LegacyApp URLs should be available
through this `LegacyController` wrapper.  Let's continue by
[wrapping a CLI script]({% post_url legacy-php-symfony/2014-07-24-wrap-cli-script %}).
