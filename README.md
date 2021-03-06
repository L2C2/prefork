# prefork

PHP class for pre-loading heavy PHP apps before serving requests.

A typical PHP web app loads itself every time there is a request. This
can be fast enough while the app is small. However, apps become
noticeably slower as they grow larger in terms of files, lines of code
and complex initialization.  This duplication of work wastes time and
money.

A Prefork PHP web app loads itself much less often. It reuses the
loaded code and initialized variables many times, reducing page load
times and increasing the capacity of each server to handle more
requests.

## Requirements

* **PHP 5.4** or higher due to `http_response_code()`
* **libevent 2** with
    [PECL package](http://pecl.php.net/package/libevent) >= 0.1.0
* **headers_list()** (PHP-CGI or patched PHP-CLI)

There are two ways you can satisfy the `headers_list()` requirement.
Probably the easiest way is to run Prefork in the PHP-CGI executable.
If this is a problem you can compile PHP-CLI with a small patch which
enables collecting the `header()` calls. (Normally the CLI SAPI sends
all `header()` calls to a black hole to save a wee bit of memory; the
patch simply sets this behavior back to the default so that Prefork
can access any headers set by the app.)

## Setup

#### index.php

    <?php
    // This script takes the place of your app's index.php
    require 'prefork.php';
    Prefork::use_ini_file( 'prefork.ini' );
    if ( Prefork::start_agent() )
        exit; // Prefork worked!
    // Otherwise load and run the app normally
    require 'my-prefork-app-loader.php';
    run_my_app();
    exit;


#### my-prefork-service.php

    <?php
    // This script is the daemon
    require 'prefork.php';
    Prefork::use_ini_file( 'prefork.ini' );
    Prefork::become_service();
    // Workers reach this code
    require 'my-prefork-app-loader.php';
    Prefork::fork();
    // Interns reach this code
    run_my_app();
    exit;


## The several kinds of processes

**Agents**: Whereas a typical web app is loaded and run for requests
to index.php, Prefork apps are not. Instead, index.php loads only a
light-weight Agent that forwards requests to the service and serves
responses back to clients. If the service is unavailable the app can
still be loaded to serve the request.

**Service**: When you start the service it spawns as many worker
processes as you configured. Then the service listens for requests on
its front-end socket and responses on its back-end, shuttling those
between agents and workers.

**Workers**: These processes load the app code and initialize any data
structures that are constant between requests. Then they connect to
the service to offer to handle a request. When a request arrives, the
worker spawns an intern to handle it. In single-intern mode, a worker
will wait until the intern exits before connecting to the service to
extend another offer. Otherwise, the worker offers to take another
request as soon as system resources allow.

**Interns**: These processes are exact copies of their parents
(Workers) so their environments are perfectly set up for running the
app to generate a response. They have the fully loaded app, the
request variables and a way to return the response to the Agent via
the Service. If an Intern exits abnormally then its Worker process
sends an error response.

## Understanding prefork vs. postfork

* Code before `Prefork::fork()` is run only once for each worker.
* Code after `Prefork::fork()` is run for each request processed.

There is an important limitation on what can be done in a prefork app
loader. Generally, don't rely on any conditions or values set by the
request; you may only rely on conditions and values that are constant
across all requests.

Superglobal request variables (`$_GET`, `$_POST`, `$_COOKIE`,
`$_SERVER`, etc.) are not ready until after `Prefork::fork()`. You
MUST NOT access them earlier.

Certian resource types are reused by each of one worker's interns in
turn. By way of example, MySQL and Memcache connections are typically
reused, but with no concurrency due to the limit of one intern per
worker. Thus if you use any persistent resources like these, you
should understand that their state is as clean as it was left by the
last intern. If they must be cleaned prior to use in the next request,
your app may use `postfork_callback`.

## `prefork_callback` and `postfork_callback`

If you assign a callable to either of these, your callback will be run
at the appropriate time. This is not strictly necessary because you
can put the same code before and after your call to
`Prefork::fork()`. The callback options add power for apps which need
it. For example, an app that calls `Prefork::fork()` in several
different places can use the callbacks instead of duplicating code
before and after each call.

## Call `Prefork::fork()` in several different places?

A good example is WordPress. This app has two major modes. In the
first mode, only one "site" is installed so the theme and plugins are
pre-determined. The second mode allows multiple "sites" to be served,
differentiated by variables such as the requested host and path. Thus
single-site WordPress can be forked much later in the app than
multi-site WordPress. (But be aware that loading a theme before
forking means you have to make a special accommodation for theme
preview mode. This could mean index.php inspects request variables
prior to becoming an agent, and loads the app normally for theme
preview requests.)
