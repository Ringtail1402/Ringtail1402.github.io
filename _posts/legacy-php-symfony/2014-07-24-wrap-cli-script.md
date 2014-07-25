---
layout: post-en
category : "en"
title: "Symfony and Legacy Code: Wrapping legacy console scripts"
tags: [php, symfony2, symfony-legacy]
---
{% include JB/setup %}

> We have already [examined how to wrap an arbitrary legacy URL]({% post_url legacy-php-symfony/2014-07-23-wrap-legacy-url %})
> in our Symfony app. Turns out, many apps also have some CLI (console) scripts, used to run cron jobs,
> or for some low-level configuration, or whatever.  What are we going to do with them?

Well, pretty much exactly what we did with URLs: instead of a catch-all `LegacyController`,
write a catch-all `LegacyCommand`.  This is going to be a bit easier as we do not
need to mess with HTTP headers and responses for this.

The only catch is that we probably do not want to have our legacy script output
wrapped in `ob_start()`/`ob_get_clean()`.  So we'll add an argument to our
`LegacyBridge#enterLegacyApp()` method to make output buffering optional:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle;

class LegacyBridge
{
    /* ... */

    public function enterLegacyApp($filename, Request $request = null, $skip_buffering = false)
    {
        /* skip setup */

        if (!$skip_buffering) {
            ob_start();
        }

        require $path;

        if (!$skip_buffering) {
            $response = ob_get_clean() ?: '';
        } else {
            $response = '';
        }

        return $response;
    }
}
{% endhighlight %}

Legacy PHP scripts probably expect command line parameters to be present in
`$_SERVER['argv']`.  We will need to hide our wrapper command from this array
just as we faked `$_SERVER['PHP_SELF']`/`$_SERVER['SCRIPT_NAME']` for URLs.

> Taking this into consideration, the actual command code is going to be
> straighforward:

{% highlight php %}
<?php

namespace Enotogorsk\LegacyApp\LegacyInteropBundle\Command;

/* "use" declarations skipped */

class LegacyCommand extends Command
{
    private $legacy;

    public function __construct(LegacyBridge $legacy)
    {
        parent::__construct();
        $this->legacy = $legacy;
    }

    protected function configure()
    {
        $this
            ->setName('legacy:command')
            ->setDescription('Executes a legacy PHP script')
            ->addArgument('filename', InputArgument::REQUIRED, 'Filename of script to run, e.g. admin/cron.php')
            ->addArgument('arguments', InputArgument::IS_ARRAY, 'Original arguments');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $filename = $input->getArgument('filename');

        // emulate plain command line
        $_SERVER['argv'] = array_merge([$filename], $input->getArgument('arguments'));

        // execute
        $this->legacy->enterLegacyApp($filename, null, true);
    }
}
{% endhighlight %}

I prefer to declare commands as dependency-injected services, same as with controllers, hence getting
`LegacyBridge` in constructor instead of inheriting from `ContainerAwareCommand`
and getting it directly from container.

So how are we going to call our CLI scripts now?

    $ php ./app/console legacy:command scripts/legacy-script.php arg1 arg2 arg3

Instead of:

    $ php scripts/legacy-script.php arg1 arg2 arg3

The `./app/console legacy:command` part may be hidden by declaring a shell alias
or a one-line wrapper script.