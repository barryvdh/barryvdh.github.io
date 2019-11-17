---
layout:     post
title:      A Symfony approach to Form handling in Laravel
date:       2019-11-17 12:00:00
excerpt:    Form handling can be done in different ways in Laravel. This post shows how to do it 'Symfony style' using the Symfony Forms Component. 
categories: laravel symfony forms
---

Form handling can be done in different ways in Laravel. This post shows how to do it 'Symfony style' using the [Symfony Forms Component](https://symfony.com/doc/current/forms.html).

## The popular Laravel approach
The most popular approach in Laravel seems to be:
 - Build the HTML manually or using a 'Form builder' / macro / helpers that generate the HTML for you.
 - Validate and handle the input in the Controller (possibly using Form Requests)
 
This does seem pretty straightforward, especially for sites with just a few unique, custom-built forms. Writing the HTML manually does leave you with a lot of repeated code:

Example from [Laravel Auth stubs](https://github.com/laravel/ui/blob/v1.1.1/src/Auth/bootstrap-stubs/auth/register.stub)

```html
<div class="form-group row">
    <label for="name" class="col-md-4 col-form-label text-md-right">{{ __('Name') }}</label>

    <div class="col-md-6">
        <input id="name" type="text" class="form-control @error('name') is-invalid @enderror" name="name" value="{{ old('name') }}" required autocomplete="name" autofocus>

        @error('name')
            <span class="invalid-feedback" role="alert">
                <strong>{{ $message }}</strong>
            </span>
        @enderror
    </div>
</div>
```
 
It's still very readable, but for each input you need to:
 - Render the input itself
 - Check error messages and adjust the validation state / render messages
 - Check the old input (and perhaps default input)
 - Optionally: Add input attributes to match the validation rules
 - Validate the fields
 - Update/create your model with the for the input you specify
 
 If you have a large forms, this could become cumbersome. Besides creating all the HTML (which a form builder could help you with), 
 you also need to keep the inputs in-sync with the validation rules and handling of the input itself.
 
 ## The Symfony Forms approach
 Symfony Forms isn't just a 'form builder' in the sense that is just generates HTML for your forms. 
 It also handles validation (and setting the error messages), securely updating your model (only for the fields you defined) and old input.
 
