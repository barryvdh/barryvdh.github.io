---
layout:     post
title:      Unobtrusive JavaScript with jquery-ujs and Laravel
date:       2015-04-11 16:45:00
excerpt:    This blogs explains how you can simplify some CRUD action using jquery-ujs, just like it's used in Ruby on Rails.
categories: laravel jquery
---

[jquery-ujs](https://github.com/rails/jquery-ujs) is a script, originally created for Ruby on Rails, to simplify common actions and make it easier to use resourceful routes. Even though it was created for Rails, it works perfectly with Laravel. It is described like this on the readme:
 
> This unobtrusive scripting support file is developed for the Ruby on Rails framework, but is not strictly tied to any specific backend. You can drop this into any application to:

> - force confirmation dialogs for various actions;
> - make non-GET requests from hyperlinks;
> - make forms or hyperlinks submit data asynchronously with Ajax;
> - have submit buttons become automatically disabled on form submit to prevent double-clicking.
 
As many of you probably realized, Laravel took some inspiration from Rails and the [RESTful Resource Controllers](http://laravel.com/docs/5.0/controllers#restful-resource-controllers) in Laravel are _very_ similar to the [routing in Rails](http://guides.rubyonrails.org/routing.html#crud-verbs-and-actions). They also share the same concept for 'faking' the request method using a hidden `_method` field to make `DELETE`, `PUT` or `PATCH` requests. So while the script name (rails.js) and it's description might make it sound like it's mainly useful for Rails, we can actually use this script in Laravel apps without any modification. [This article](https://robots.thoughtbot.com/a-tour-of-rails-jquery-ujs) gives a good tour of the functionality, but focuses on Rails. So in my blog post, we will focus on use with Laravel.

### Unobtrusive JavaScript
`jquery-ujs` presents itself as 'unobtrusive'. From the previously linked article:

> The UJS in jquery-ujs stands for [unobtrusive JavaScript](http://en.wikipedia.org/wiki/Unobtrusive_JavaScript). This is a rather broad term that generally refers to using JavaScript to progressively enhance the user experience for capable browsers without negatively impacting clients that do not support or do not enable JavaScript.

> jquery-ujs wires event handlers to eligible DOM elements to provide enhanced functionality. In most cases, the eligible DOM elements are identified by [HTML5 data attributes](http://ejohn.org/blog/html-5-data-attributes/).

Most of the `jquery-ujs` functionality works by just adding a `data-*` attribute to an element. This means that for a lot of features, you don't have to register any bindings in jQuery itself.

### Installation
You can install it with bower, mix it up with elixir or just copy the [src/rails.js](https://github.com/rails/jquery-ujs/blob/master/src/rails.js) file to your project and include it like any other script. Some functions will work right away, but for some you need to account for CSRF protection.

#### Handling CSRF Protection
In [my last blog post](http://barryvdh.nl/laravel/2015/02/21/csrf-protection-in-laravel-explained/) I explained how CSRF protection works in Laravel and also showed how to use it with Javascript. Luckily jquery-ujs handles most of the work already. All you have to do is add a `csrf-token` and `csrf-param` meta-tag to the head of your document. 

```
<meta name="csrf-token" content="<?= csrf_token() ?>" />
<meta name="csrf-param" content="_token" />
```

This will set a `X-CSRF-Token` header on each XHR-request (for `data-remote`) and add a hidden `_token` field for the `data-method` requests.
### Examples
Many examples are given in the [wiki from the repository](https://github.com/rails/jquery-ujs/wiki/Unobtrusive-scripting-support-for-jQuery). A few highlights:

#### Confirm
Before submitting a form or following a link, show a confirmation dialog, so you don't accidentally delete the wrong item for example.

```
<form data-confirm="Are you sure you want to submit?">...</form>
```

#### Disable with
When you need to perform an action and the response take a little while, users may be tempted to click the button again. The `disable-with` function simply disables the submit button once it's clicked, to prevent repeated actions.

```
<input type="submit" value="Save" data-disable-with="Saving...">
```

#### POST/PUT/DELETE through links
As you probably know, Laravel uses a `DELETE` request to perform the `destroy()` method on the resource controller. This usually means you have to create a form with a hidden `_method` field set to `DELETE`. With jquery-ujs you can simply create a link with a `data-method` attribute. This get's transformed into a `DELETE` request by jquery-ujs automatically.

```
<a href="..." data-method="delete" rel="nofollow">Delete this entry</a>
```

And of course you can combine multiple functions, to verify you actually need to remove that entry.

#### Making ajax requests
With the `data-remote` option you can make a form (or link) perform it's action as an ajax request in the background.

```
 <form data-remote="true" action="...">
      ...
    </form>
```

If you want to handle the output, you can attach listen to [custom events](https://github.com/rails/jquery-ujs/wiki/ajax). Within Laravel, you can detect whether it's a regular or XHR request.

```
<a href="<?= route('photo.delete', 1) ?>" class="destroy-btn" data-method="delete" data-remote="true">Delete this entry</a>
```

```
$('.destroy-btn').bind('ajax:success', function(e, data, status, xhr){
    alert("Deleted: "+data);
});
```

```
public function destroy($id) {
    if (\Request::ajax()) {
        return $id;
    }
    return redirect()->back();
}
```

For more options, like extra parameters, data type etc., see [the wiki](https://github.com/rails/jquery-ujs/wiki).

### That's about it
I hope this gives you a good idea of how jquery-ujs could be useful for you. I use it in a lot of project to avoid having to write the same boilerplate javascript handlers and just because it's easy to use. If you have a better approach or other feedback, let me know!

