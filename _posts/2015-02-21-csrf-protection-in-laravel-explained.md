---
layout:     post
title:      CSRF Protection in Laravel explained
date:       2015-02-21 15:40:00
excerpt:    In this blog we take a closer look into CSRF protection in Laravel. We compare the difference between the CSRF filter in Laravel 4 and the current VerifyCsrfToken middleware in Laravel 5.
categories: laravel
---

In this blog we take a closer look into CSRF protection in Laravel. We compare the difference between the CSRF filter in Laravel 4 and the current VerifyCsrfToken middleware in Laravel 5.

## Why do we need CSRF protection?

Laravel has CSRF-protection enabled by default. So even if you don't know what CSRF is, or why you need to protect your apps from it, you'll probably run in to an `Illuminate\Session\TokenMismatchException` pretty fast and realize you have to add that hidden `_token` field with the `csrf_token()` value..

But not everybody knows exactly what it protects your app from. So if you don't, go read [this article from Anthony Ferrara  (ircmaxell)](http://blog.ircmaxell.com/2013/02/preventing-csrf-attacks.html). Here is a short excerpt:

> __Request Forgery. [..] From Another Site__: This happens when an attacker on another site (one they have control over) submits a request to the target site. The browser will send cookies to the target site, so if a user has permissions on the remote site, the action will be performed. No defense that we mentioned so far will effectively protect against this. Because it requires two sites to execute, it's called a Cross-Site-Request-Forgery (CSRF).

So for example, if you are logged in on Facebook, a hacked site could run some Javascript to make you post something under your Facebook account. Of course we don't want that to happen, so we need protection for it. And here is where the CSRF Middleware helps us.

Because of the [Same-Origin security policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#Cross-origin_network_access) in browsers, JavaScript cannot read responses from a different domain, but write requests can be executed. So even though the attacker doesn't get the response, the request is still executed.

A common solution is to add a CSRF token to our form, which is generated on each request and validated on POST/PUT/DELETE requests. Because the token cannot be read, attackers can't make those requests any more.

## CSRF in Laravel

In Laravel 4.0, there was a pre-defined CSRF filter in your app/filters.php that looked like this:

```php
Route::filter('csrf', function() {
	if (Session::token() != Input::get('_token')) {
		throw new Illuminate\Session\TokenMismatchException;
	}
});
```

This wasn't applied by default, but you could easily add this to the routes you want to protect. But what was wrong with this?

### Strict typing
> [blog.laravel.com](http://blog.laravel.com/csrf-vulnerability-in-laravel-4/): Note that the token comparison has been changed from a != comparison to a !== comparison. This will prevent specially crafted JSON requests from bypassing the filter.

Because of the weak-typing and using `json_decode()` for JSON requests, you could pass `{'_token':true}` as JSON data and it would still match. That's a pretty easy (but important!) fix, but as [lasselehtinen and ircmaxell noted](https://github.com/laravel/laravel/commit/ba0cf2a1c9280e99d39aad5d4d686d554941eea1), it's still not perfect.

### Timing safe comparison

While a timing attack for CSRF tokens is probably more theoretical, it would be possible for attackers to guess the token by repeating the same request many times and comparing the time it takes to return. Because a regular string comparison stops when it finds a character that doesn't match, you could find a difference between different tokens. Very simplified example:

```php
$token = 'abcdef';
$token === 'abaaaa'; // 1ms
$token === 'abbaaa'; // 1ms
$token === 'abcaaa'; // 2ms, takes longer so 'abc' probably matches.
```

You can find more information in the [PHP RFC for timing attacks](https://wiki.php.net/rfc/timing_attack) which added the `hash_equals()` method in PHP5.6. Symfony Security Core provides a pure PHP alternative with [StringUtils::equals()](https://github.com/symfony/security-core/blob/a2403347bc6f25a2b06c4ffc6977175371b2e302/Util/StringUtils.php#L28-L65)

```php
use Symfony\Component\Security\Core\Util\StringUtils;
Route::filter('csrf', function() {
	if ( ! StringUtils::equals(Session::token(), Input::get('_token')))
	{
		throw new Illuminate\Session\TokenMismatchException;
	}
});
```

This was [rejected for Laravel 4](https://github.com/laravel/laravel/pull/3126) but [added in Laravel 5](https://github.com/laravel/framework/pull/6385).

### CSRF Middleware in Laravel 5

Laravel 5 enables the [VerifyCsrfToken middleware](https://github.com/laravel/framework/blob/5.0/src/Illuminate/Foundation/Http/Middleware/VerifyCsrfToken.php) by default for all requests, which is a good thing. It's a bit more advanced, and does the following:

1. Check if the request is a reading request (HEAD, GET, OPTIONS). If so, skip the check.
2. Match the token from the `_token` input or from the headers.
3. Add a cookie with the token to each request.

This makes the CSRF check a lot more flexible. You don't have to remember where to add you filters, just make sure that every form has a `_token` field. Because of #2 and #3, it will work with Ajax request without having to modify the core filter.

> Note: This reminds us again that GET requests should never change state. The CSRF middleware assumes that it doesn't need to check GET (or HEAD/OPTIONS) requests, because they should be safe to execute.

### Checking the headers

At first, only the `X-XSRF-TOKEN` was checked. This used the [Angular convention](https://docs.angularjs.org/api/ng/service/$http#cross-site-request-forgery-xsrf-protection) that the token could be read from the `XSRF-TOKEN` cookie. If Angular detects that cookie, it [adds the token to all XHR requests](https://github.com/angular/angular.js/blob/5da1256fc2812d5b28fb0af0de81256054856369/src/ng/http.js#L1072).

```Javascript
var xsrfValue = urlIsSameOrigin(config.url)
    ? $browser.cookies()['XSRF-TOKEN']
    : undefined;
if (xsrfValue) {
  reqHeaders['X-XSRF-TOKEN'] = xsrfValue;
}
```

While this does work great for Angular, it has a slight problem: Because the cookies in Laravel are always encrypted, the token from the cookie needs to be decrypted before it can be compared. This is not a problem for Angular, but it is a problem if you want to set the header manually for your own JavaScript requests.

In Laravel 5.0.6, [a patch landed](https://github.com/laravel/framework/pull/7528) which added support for a plain text `X-CSRF-TOKEN` header.

```php
protected function tokensMatch($request) {
    $token = $request->input('_token') ?: $request->header('X-CSRF-TOKEN');
    if ( ! $token && $header = $request->header('X-XSRF-TOKEN'))
    {
        $token = $this->encrypter->decrypt($header);
    }
    return StringUtils::equals($request->session()->token(), $token);
}
```

You could now, for example, simply add a meta-tag to your `<head>` section, read it with jQuery and set the XHR header:

```
<meta name="csrf-token" content="<?php echo csrf_token() ?>" />

$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

This will set the token header for all your jQuery requests. [jQuery UJS](https://github.com/rails/jquery-ujs) also follows this meta-tag convention for example.

### Modifying the middleware

I would suggest that you try to avoid writing your own logic for the CSRF handler. As you can see above, there are a lot of things to consider. If some bug is found with the VerifyCsrfToken middleware, it can now be fixed upstream. If you have your own filter, like in Laravel 4, it won't be updated automatically.. But since [this commit to laravel/laravel](https://github.com/laravel/laravel/commit/c3e3d9dc4b8a4f6f52f1f89233f2a1d19011fc24), you can easily override certain methods in your own app.
I you don't want to use CSRF at all, you can still just remove the VerifyCsrfToken from your middleware list.

### Comments?

I hope this clears up some confusion about CSRF protection and how Laravel handles it. If you spot any mistakes or have questions/improvements, feel free to open an issue on [my blog repository](https://github.com/barryvdh/barryvdh.github.io)!
