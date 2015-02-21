---
layout:     post
title:      Using StackPHP middleware in Laravel5
date:       2015-02-18 20:57:00
excerpt:    Laravel 4 was compatible with StackPHP middleware, but Laravel 5 uses a new way to handle middleware. This blog explains the differences and shows a way to still use Stack middleware.
categories: laravel
---

__TLDR__; Want StackPHP middleware in Laravel 5.0? Try [barryvdh/laravel-stack-middleware](https://github.com/barryvdh/laravel-stack-middleware)

### Middleware and Laravel 4

In version 4.1, Laravel introduced compatibility with [StackPHP middleware](http://stackphp.com/). As Laravel uses the Symfony [HttpFoundation](http://symfony.com/doc/current/components/http_foundation) and the Application class implements the [HttpKernelInterface](https://github.com/symfony/symfony/blob/master/src/Symfony/Component/HttpKernel/HttpKernelInterface.php), it made sense to support this. This made it easy to use middlewares from StackPHP, just like in Symfony/Silex or other frameworks using the HttpKernelInterface. See [this article from Chris Fidao](http://fideloper.com/laravel-http-middleware) for more information.

### Middleware in Laravel 5

In Laravel 5, a lot of things changed. And with those changes, Laravel also removed the support for StackPHP middleware and introduced its [own middleware contract](https://github.com/illuminate/contracts/blob/5.0/Routing/Middleware.php). Matt Stauffer explains the nicely on [his blog](http://mattstauffer.co/blog/laravel-5.0-middleware-filter-style) but (spoiler alert) here is the last paragraph:

> Not only that, but middleware is just one more way of working with your request in a way that is both powerfully effective in your Laravel apps, but plays nicely elsewhere else. The Laravel 5.0 middleware syntax isn't perfectly compatible with StackPHP syntax, but if you structure your request/response stack along the organizational structure of middlewares it's a further work in the direction of separation of concerns--and modifying a Laravel Middleware to work in a separate, StackPHP-style syntax, would take minimal effort.

### Converting Stack middleware to Laravel middleware

As the quote from Matt's blog suggests, it isn't that hard to convert your Stack middleware to the new Laravel 5 style middleware. Both basically do the following:

 - Take the Request object and do something with it (optionally)
 - Either return a Response or let the next middleware handle it further (until one returns a Response)
 - Optionally do something with the Response object, before returning it.
 
So the input is a Request, the output a Response. That hasn't changed in the new Laravel middleware. Take this basic example with the HttpKernelInterface

```php
public function handle(SymfonyRequest $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true)
{
    // Do something with $request
    $response = $this->app->handle($request, $type, $catch);
    // Do something with $response
    return $response;
}
```

And convert it to Laravel Middleware:

```php
public function handle($request, Closure $next)
{
    // Do something with $request
    $response = $next($request);
    // Do something with $response
    return $response;
}
```

### Using Stack middleware in Laravel 5

So sure, it's possible to convert your middleware. And that is probably the easiest solution when you are upgrading. But what about the [middlewares from StackPHP](http://stackphp.com/middlewares/)? For example, there are some simple ones that just add some headers but some (like HttpCache) are a bit more complex. And the idea of StackPHP was to be able to share your middleware, not re-invent the wheel. So, how to use them in Laravel 5?

Obviously, the two interfaces are conflicting. Both use the `handle()` method but with different arguments. So we could create a Laravel middleware that wraps the Stack middleware and calls its `handle()` method with the correct arguments.

```php
public function handle($request, Closure $next)
{
    return $this->middleware->handle($request);
}
```

But any Stack middleware needs a HttpKernel as first constructor argument (by convention), so what do we pass as kernel? The Laravel Application class still implements the HttpKernelInterface, but that just calls the middleware stack so that will crash your application. So we also need to create a wrapper for the HttpKernelInterface. And that wrapper needs to be able to call the `$next` closure.

```php
public function handle(Request $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true)
{
    $next = ??;
    return $next($request);
}
```

### laravel-stack-middleware Package

So, to solve the chicken-egg question, I created two wrappers. The [ClosureMiddleware](https://github.com/barryvdh/laravel-stack-middleware/blob/master/src/ClosureMiddleware.php) that receives the `$next` closure and passes it to the  [ClosureHttpKernel](https://github.com/barryvdh/laravel-stack-middleware/blob/master/src/ClosureHttpKernel.php). The Kernel then receives the `$request` on its `handle()` method and calls the `$next` closure it just received. So we just do something like this:

```php
// ClosureMiddleware
public function handle($request, Closure $next) {
    $this->kernel->setClosure($next);
    return $this->middleware->handle($request);
}

// ClosureHttpKernel
public function handle(Request $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true) {
    $closure = $this->closure;
    return $closure($request);
}

// Wrapping it up together
$kernel = new ClosureHttpKernel();
$stackMiddleware = new Some\Stack\Middleware($kernel, $param1, $param2);
$middleware = new ClosureMiddleware($kernel, $stackMiddleware);
```

That all sounds a bit cumbersome so I've create a simple package for it: [barryvdh/laravel-stack-middleware](https://github.com/barryvdh/laravel-stack-middleware)
It's still pretty new and not very well tested, but it does seem to do its job. I've also used this in my [HttpCache package](https://github.com/barryvdh/laravel-httpcache) to wrap the Symfony HttpCache middleware.
The readme explains more, but the previous example would become:

```php
public function boot(StackMiddleware $stack) {
    $stack->bind('DoSomeMiddlewareStuff', 'Some\Stack\Middleware', [$param1, $param2]);
}
```

That binds a new Middleware to the App container under the name `DoSomeMiddlewareStuff` so you can add it to your $middleware array in your app Kernel.php

### Way forward

So Taylor probably had good reasons to change to this new middleware style (less dependend on HttpFoundation?) but you still need to make some assumptions about what kind of Request or Response everyone is expecting. Hopefully [PSR-7 will bring a unified standard for this](https://mwop.net/blog/2015-01-08-on-http-middleware-and-psr-7.html), so middlewares can be easily shared between frameworks. As long as we follow the same principles and request object, it hopefully shouldn't be too hard to make it compatible with different frameworks..

### Comments?

If you think I'm saying/doing something wrong here or with my wrapper for StackPHP middleware, please let me know! You can leave me a comment on Twitter or create an issue in the [laravel-stack-middleware repo](https://github.com/barryvdh/laravel-stack-middleware/issues) or the [issue tracker from my blog](https://github.com/barryvdh/barryvdh.github.io/issues).
