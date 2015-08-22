---
layout:     post
title:      Comparing Blade and Twig templates in Laravel
date:       2015-08-22 22:00:00
excerpt:    In my company, we use Twig instead of Blade for our Laravel projects. I know there are a lot of developers that also prefer Twig over Blade. So the question 'Why choose Twig over Blade?' often pops up. The reason is usually just a matter of preference, but in this post we're going to compare the Blade and Twig templating systems. 
categories: laravel twig
---

In my company, we use Twig instead of Blade for our Laravel projects. I know there are a lot of developers that also prefer Twig over Blade. So the question 'Why choose Twig over Blade?' often pops up. The reason is usually just a matter of preference, but in this post we're going to compare the Blade and Twig templating systems. 

## TLDR; Spoiler alert
Both Blade and Twig provide the most important features; template inheritance, sections, escaping output and clean syntax.
Blade provides a fast and simple syntax, but doesn't add (much) extra functionality.
Twig takes it a step further and adds an extra layer to provide more security and added features.
The choice mostly depends on your personal preference. If you mostly develop for Laravel, Blade would probably be good. If you also use a lot of other frameworks, Twig might be a better fit.

## About Blade
Blade is the default template engine for Laravel (since Laravel 2 in 2011), so I'm assuming you know about it. The syntax is originally inspired by the [ASP.net Razor syntax](http://www.asp.net/web-pages/overview/getting-started/introducing-razor-syntax-(c)) and provides a cleaner way to write your templates. But the syntax is just 1 part, the main benefit of using Blade instead plain PHP is to make it easier to re-use templates and split templates.

From [the Laravel 5.1 docs](http://laravel.com/docs/5.1/blade):
> Blade is the simple, yet powerful templating engine provided with Laravel. Unlike other popular PHP templating engines, Blade does not restrict you from using plain PHP code in your views. All Blade views are compiled into plain PHP code and cached until they are modified, meaning Blade adds essentially zero overhead to your application.
> [..]
> Two of the primary benefits of using Blade are template inheritance and sections. 

Example:
```
@extends('layouts.master')

@section('content')
  @foreach ($users as $user)
      <p>This is user {{ $user->id }}</p>
  @endforeach
@endsection
```

Basically, Blade syntax is a thin wrapper for PHP, to provide a clean syntax. Everything you can do with PHP, you can do with Blade (and vice versa). You can also easily mix plain PHP in your Blade templates.

## About Twig
Twig is created by Fabien Potiencer, for reasons he describes in his [blogpost announcing Twig](http://fabien.potencier.org/templating-engines-in-php.html). It's included in the default Symfony2 installation, can be used stand-alone and recently has seen some adoptions in other projects. [Drupal 8 will use Twig as template engine](https://www.drupal.org/theme-guide/8/twig) and Magento 2 was going to use it, but [decided to drop it](https://github.com/magento/magento2/issues/370) a while back. Twig support was also considered to be added in Laravel by default, but was eventually removed before the final release.

From [the site](http://twig.sensiolabs.org/):
> * **Fast**: Twig compiles templates down to plain optimized PHP code. The overhead compared to regular PHP code was reduced to the very minimum.
> * **Secure**: Twig has a sandbox mode to evaluate untrusted template code. This allows Twig to be used as a template language for applications where users may modify the template design.
> * **Flexible**: Twig is powered by a flexible lexer and parser. This allows the developer to define its own custom tags and filters, and create its own DSL.

Example:
```
{% raw  %}
{% extends "layouts.master" %}

{% block content %}
    {% for user in users %}
        <p>This is user {{ user.id }}</p>
    {% endfor %}
{% endblock %}
{% endraw  %}
```

Twig is sandboxed. You can't just use any PHP function by default, you can't access things outside the given context and you can't use plain PHP in your templates. This is done by design, it forces you to seperate your business logic from your templates. 

## Twig in Laravel

Together wit Rob Crowe, I've built a TwigBridge: https://github.com/rcrowe/TwigBridge
This offers Twig support in Laravel with all the things you would expect in Laravel: 
 - Loading view using the Laravel View factory (`view('template')`)
 - Using events for View Composers
 - Access to common functions/filters (input, auth, session etc)
 - Facade support
 
So it's easy to get started and give Twig a try :)
 
## Blade vs. Twig

So there are a few similarities here:

- Both compile to plain PHP, for better performance
- Both provide a syntax to easily output variables
- Both have control-structures (if/for/while etc)
- Both have escaping mechanism for safe output
- Both have template-inheritance and sections

There are also a few differences, that are easy to spot when you look at the examples.

- Different syntax for control structures
- Different handling of escaping
- Different handling of variables and variable access
- Different ways to add functionality
- Different perspective on security

We'll take a quick look through these differences, so you can choose yourself what you like best.

### Outputting variables

Outputting variables is probably the most common thing in your templates, so it should be easy. But more importantly, it should be safe. Luckily, since Laravel 5 Blade has sane defaults: `{{ $var }}` shows escaped content, `{!! $var !!}` raw output. The same goes for Twig, by default `{{ var }}` is escaped, `{{ var |raw }}` is raw.

Laravel gives you the option to change the tags and Twig gives you the option to change the default escaping. Both are probably not very smart in most cases, because it can be unpredictable for other developers.

Besides escaped or raw, Twig gives you the option to use different escaping methods, eg. for JS or HTML attributes. This can also be configured for an entire chunk of code.

```
{% raw  %}
{{ user.username|e('css') }}

{% autoescape 'js' %}
    Everything will be automatically escaped in this block (using the JS strategy)
{% endautoescape %}
{% endraw  %}
```

You probably noticed the `|` character. Those are used for `filters`. Filters can tweak the output. They are not very much different then functions, but they might be easier to read and can be combined. Example: `{{ var | striptags | upper }}`.

### Accessing attributes

Twig makes it easy to access variables using the dot notation. This can be used on either a object or array. In Blade, it's the same as plain PHP.

```
$user->name --> user.name
$user['name'] --> user.name
```

### Control structures

In Blade, most control structures are simply replaced by their PHP equivalent during compilation, as you can see in [the code](https://github.com/laravel/framework/blob/5.1/src/Illuminate/View/Compilers/BladeCompiler.php). Some structures are a tiny bit more complicated. `forelse` doesn't actually exist, so that adds a few more lines.

Simplified example:
```php
@forelse ($user in $user)
    @if(!$user->subscribed)
        {{ $user->name }}
    @endif
@empty
    <p>No users found</p>
@endforelse
```

```php
<?php $empty = true; foreach($user in $user): $empty = false; ?>
    <?php if (!$user->subscribed) : ?>
        <?php echo e($user->name); ?>
    <?php endif; ?>
<?php endforeach; if ($empty): ?>
    <p>No users found</p>
<?php endif; ?>
```

In Twig, the controle structures are called `tags`. They are compiled by a Lexer and can be a bit more complicated. For example, the `for` tag adds a `loop` variable to the context, so you can access the current loop state; `loop.first`, `loop.last`, `loop.index` etc. This makes it just a bit cleaner then doing it yourself. The `if` tags makes it possible to read more like a sentence, instead of just a statement.

```
{% raw  %}
{% for user in users %}
    {% if not user.subscribed %}
        {{ user.name }}
    {% endif %}
{% else %}
    <p>No users found</p>
{% endfor %}
{% endraw  %}
```

### Template inheritance and sections

Template inheritance and sections are pretty much the same. It's just different syntax. See the example from the Laravel docs, vs Twig syntax:

```
<!-- layouts/master.blade.php -->
<html>
    <head>
        <title>App Name - @yield('title')</title>
    </head>
    <body>
        @section('sidebar')
            This is the master sidebar.
        @show
    
        <div class="container">
            @yield('content')
        </div>
    </body>
</html>

```
```
<!-- child.blade.php -->
@extends('layouts.master')

@section('title', 'Page Title')

@section('sidebar')
    @parent

    <p>This is appended to the master sidebar.</p>
@endsection

@section('content')
    <p>This is my body content.</p>
@endsection
``` 

Same result in Twig:

```
{% raw  %}
<!-- layouts/master.twig -->
<html>
    <head>
        <title>App Name - {% block title %}{% endblock %}</title>
    </head>
    <body>
        {% block sidebar %}
            This is the master sidebar.
        {% endblock %}
    
        <div class="container">
            {% block content %}{% endblock %}
        </div>
    </body>
</html>
{% endraw  %}
```

```
{% raw  %}
<!-- child.twig -->
{% extends "layouts.master" %}

{% block title %}Page Title{% endblock %}

{% block sidebar %}
    {{ parent() }}
    <p>This is appended to the master sidebar.</p>
{% endblock %}

{% block content %}
<p>This is my body content.</p>
{% endblock %}
{% endraw  %}
```

### Security and context

As stated before, Blade doesn't actually differ so much from 'plain php'. It makes it easier to properly escape variables, but it doesn't place any restrictions.
Twig on the other hand, works in a seperate context. All functions and filters calls are restricted by the functions you explicitly enable (besides the built in functions).

Both have their pros and cons. In Twig, your template designer can't easily take shortcuts. Eg. calling a query in your templates. They'll have to pass the result to the view or allow access to a certain function (or ask the backend developer). The downside is that it is more work to just call a simple function/filter sometimes.

Blade
```
@foreach(User::where('active')->get() as $user)
  {{ $user->name }}
@endforeach
```

This isn't exactly possible in Twig. You either pass the result to the view (in your controller or view composer), or if you must, call the query on a User instance.

```
{% raw  %}
{% for user in model.where('active').get() %}
  {{ user.name }}
{% endfor %}
{% endraw  %}
```

This also means that you can't just use Facades. In your TwigBridge, we've made it an option to just add your facades to the list in the configuration. `Auth::check()` --> `Auth.check()`

Twig also includes a Sandbox mode, which can create a safe context, eg. to let users edit their templates, with limited access to functions/variables. These templates can also be retrieved from a database.

### Extending functionality

Extending the `directives` in Blade is simple, but limited to replacing a tag with other lines.

```php
<?php
// @upper($var)
Blade::directive('upper', function($expression) {
    return "<?php echo strtoupper{$expression}; ?>";
});
```

In Twig, writing functions and filters is very easy. But [writing custom tags](http://twig.sensiolabs.org/doc/advanced.html) is a lot harder, you usually don't need to do that. Above example could be just a simple filter:

```php
<?php
// {{ var | upper }}
$filter = new Twig_SimpleFilter('upper', function ($string) {
    return strtoupper($string);
});
```

Both engines also support macros, as you can read in the docs.

### Performance

I haven't actually tested this, but I would guess that Blade is slightly faster, but only because the Twig compiled code is a bit more complicated, because of extra functionality. But they are both compiled to plain PHP, so the performance difference can probably be neglected.

### Other differences

There are some small syntaxical and behavioral differences, that probably don't have a big impact on your choice. Twig has some niceties in custom operators, some math/expressions stuff, handy built-in filters/functions etc. You'll probably get a good idea if you read through the Twig manual, including all the tags/filters etc. But by now, you've probably already made up your mind about what you're going to use ;)

## So ..?

In my personal opinion, Twig is the more mature one. It's built from the ground up as secure, with different escaping strategies, context restrictions and lexer. But sometimes it can feel a bit too restrictive, for rapid development.

Blade is just a bit simpler, but probably good enough for most developers. It doesn't impose any restrictions, so it leaves the best practices up to the developer. It does feel more 'Laravel' like in it's simplicity, so that might appeal to new Laravel users.

However, if your company also uses different frameworks (Symfony, Drupal, Slim, Yii, Kohana, etc), the Twig support could be a big PRO. Mostly it's just a matter of preference. You can build any app you like with both templating engines and probably never run into any issues.

{% endraw %}
