---
layout:     post
title:      Using FullCalendar with Laravel Livewire
date:       2020-09-24 20:00:00
excerpt:    Integrating FullCalendar with Laravel Livewire had some interesting points that I wanted to share.
categories: laravel livewire fullcalendar
share-img:  /img/livewire-fullcalendar.png

---

## Why FullCalendar?
We use [FullCalendar](https://fullcalendar.io/) in a lot of projects and I have yet to find anything that is as powerful and flexible for creating calendars. 
We've used plain FullCalendar before also the Vue integration, but lately [Livewire](https://laravel-livewire.com/) has taken my interest. 
I wanted to see if I could integrate this nicely. I'm happy with the result so far and wanted to share some of the things I encountered. 
We don't cover the basics of creating your Livewire component, but you can read about it in the [Quickstart](https://laravel-livewire.com/docs/2.x/quickstart).

[![FullCalendar and Laravel Livewire](/img/livewire-fullcalendar.png)](https://laravelplayground.com/#/snippets/062218a2-cd89-413a-b442-54b94a2056b3)
[See the final example on Laravel Playground here.](https://laravelplayground.com/#/snippets/062218a2-cd89-413a-b442-54b94a2056b3)

## Drag & Drop
The starting-point of my journey was the [Drag & Drop Demo](https://fullcalendar.io/docs/external-dragging-demo) ([Docs](https://fullcalendar.io/docs/external-dragging)) which shows the Javascript side. 
This is a common pattern for planning/scheduling for use; take a list of events and plan them. Of course the changes need to be stored Server Side, which is were Livewire comes in.

## Rendering the Calendar once
To start, we can use the same code from a CDN and make sure we include the initialisation of the FullCalendar scripts etc are OUTSIDE the livewire component, to make sure they render once.
For example by using `@stack('scripts')` in your layout file and `@push('scripts')` in your component.

[Playground example](https://laravelplayground.com/#/snippets/e4d1ca76-6ff9-4743-af8b-ee81ef65e339)

The Calendar itself gets rendered inside the component, but we want to persist is across livewire events. So we add the `wire:ignore` attribute to our Calendar container. This will let Livewire ignore the contents.

```html
<div id="calendar-container" wire:ignore>
  <div id="calendar"></div>
</div>
```

Now each time Livewire reloads, the FullCalendar is still rendered. [See the Playground](https://laravelplayground.com/#/snippets/790e0206-91a2-4a2c-9394-e53f8d18dd6f)

## Calling actions on Events
To store the events, we need to pass the data to the back-end. With Livewire this is super easy. We can use `@this.<myaction>` to call functions in PHP, straight from the Javascript FullCalendar events:

```js
{
  ...,
  eventReceive: info => @this.eventReceive(info.event),
  eventDrop: info => @this.eventDrop(info.event, info.oldEvent)
}
```

This will call the functions in your component:

```php
public function eventReceive($event)
{
    // Handle new events
}

public function eventDrop($event, $oldEvent)
{
  // Modify existing events
}
```

The $event will contain the EventObject from FullCalendar, so you can pass an ID to match it, or supply additional data.
[See the example on Playground](https://laravelplayground.com/#/snippets/80fb6377-d7ae-49b0-bd36-1ac563e52994)

## Refreshing the data
One of the benefits of FulLCalendar is the Event Source API, which makes it easy to load your events from an API. 
The benefit from Livewire is that you can reload your data easily, but we want to use the API to optimize the Event loading when required.

We can add [Dynamic Parameters to our Event Source](https://fullcalendar.io/docs/events-json-feed) to make use of the Livewire inputs if required.

```js
calendar.addEventSource( {
  url: '/calendar/events',
  extraParams: function() { 
      return {
          name: document.getElementById("selectName").value
      };
  }
});
```
The start/end time will be appended to the QueryString, together with the extraParams. So you can only load the current month/week etc.

By default, this will load only once, at the start. But we want to trigger it from our PHP Component. So we can add a listener using `@this.on()`:

```js
@this.on(`refreshCalendar`, () => {
    calendar.refetchEvents()
});
```

Now we can emit this event from inside our component, eg. when the name changes:

```php
public function updatedName()
{
    $this->emit("refreshCalendar");
}
```

Now we can decide when to reload any data, and it will be dynamically using the correct name!

[You can see the working example here](https://laravelplayground.com/#/snippets/062218a2-cd89-413a-b442-54b94a2056b3)

## Wrapping up

This only explains some basic stuff but it shows a lot of potential, for me. It removes a lot of boilerplate and simplifies the handling of changes in the Events, but moving it all to PHP.
By keeping in control of the refreshing of data, it still feels very fast, without using too much javascript.

Let me know if you have any questions or suggestions!



