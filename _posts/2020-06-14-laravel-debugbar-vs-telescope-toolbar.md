---
layout:     post
title:      Debugbar vs Telescope Toolbar
date:       2020-06-14 10:00:00
excerpt:    I developed both Telescope Toolbar and Laravel Debugbar, and many want to know the difference. Let's compare both.
categories: laravel debugbar telescope
---

## A little history

When we started using Laravel at Fruitcake, it was around the time of Laravel 3. It didn't have Composer support yet and was a bit limited, but did contain things like Eloquent and Blade templates.
One of the things it also contained was a small toolbar called 'Anbu'. Not sure who created it, but it came with Laravel by default.

(Anbu image)

When Laravel 4 came around the corner, it was pretty great with Composer support etc. There were 2 main issues we had with it.
 - PhpStorm didn't understand Facades
 - No more Anbu toolbar
 
 For the first one, I created [laravel-ide-helper](https://github.com/barryvdh/laravel-ide-helper) and for the second one, I came accross http://phpdebugbar.com/
 
 (Debugbar starred image)
 
 And just before the official release at the end of 2013, Laravel Debugbar was released, already providing most of the features you use today:
 
 (v1)
 
 During the last 7 years the layout changed to make it Laravel specific, new collectors have been added and functionality has improved (also upstream to debugbar).
 
 (v3)
 
 5 years after (November 2018), [Laravel released Laravel Telescope](https://laravel-news.com/laravel-telescope-1-0-0), giving Laravel developers a first-party tool.
 
 (telescope)
 
 During the last 7 years, the core of Debugbar didn't change and was still using a lot of jQuery. I looked into the Symfony toolbar as modern alternative a few times, but it didn't warrant such a big rewrite.
 When Telescope launched, I liked the general approach with Storage etc and Symfony Toolbar was now decoupled enough to use stand-alone. Thus [Laravel Telescope Toolbar](https://github.com/fruitcake/laravel-telescope-toolbar) was born.
 
 (telescope toolbar)
 
 ## Technology differences
 
Laravel Debugbar uses Collectors to gather data during the request cycle. At the end of the request, all data is collected, formatted and stored (usually in the storage dir, but DB can be used). For normal requests, the data is then flashed in the session and rendered on the next requests. For ajax requests the ID is passed in the headers and loaded through an EventListener on XHR requests. All gathered data is loaded and formatted directly by the Debugbar, which can cause delays for large datasets (eg. 1000+ queries). This usually isn't a big problem, but more of an indicator that you should optimize ;)
3rd party libraries are used (Font Awesome, Highlight.js, jQuery), but these are scoped to avoid interference.

Telescope Toolbar only displays data that is already gathered by Telescope. The data is always loaded async, after the page has loaded, for both normal and ajax requests. No external dependancies are used to prevent library collissions, just plain Javascript and SVG icons. The Toolbar content is rendered by PHP (Blade templates), so it doesn't free your browser when displaying a lot of data (that much), and it's easier to show just the summary and redirect to Telescope for the rest.

In short; Debugbar is both the toolbar and the detailed info, Telescope Toolbar is just the toolbar and leverages Telescope for the rest.

Debugbar:
 - More information directly visible
 - More browser heavy
 
Telescope Toolbar
 - Less browser heavy
 - Need to click through for more information.
 
 ### Features
  
 ## Debugging Queries
 
 ## Showing Ajax Requests
 
 ## Timeline
 
 ## Dumping data
 
 
 
