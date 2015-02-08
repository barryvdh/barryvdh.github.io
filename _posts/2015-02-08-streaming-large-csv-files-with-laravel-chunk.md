---
layout:     post
title:      Streaming large CSV files with Laravel chunked queries
date:       2015-02-07 22:12:29
summary:    
categories: laravel php
---

### Exporting data
When I create websites to manage data, like users, items, products etc., a common request is to have an export of the data. If this is a one-time request for plain data, you could just export it using your favorite admin-tool. But usually you want to modify some fields, add related models and make it automated so that the client can do it himself.

Luckily exporting data in PHP is pretty simple, especially if you just create a CSV file. Just look at the example from [fputcsv on php.net](http://php.net/manual/en/function.fputcsv.php):

```php
$out = fopen('php://output', 'w');
fputcsv($out, array('this','is some', 'csv "stuff", you know.'));
fclose($out);
```

### Using StreamedResponse
But in Laravel (or any other frameworks that uses Response objects), you don't want to just write to the output stream. You want to create a proper Response object, add some headers, run it through your middleware stack etc etc. Luckily the Symfony HttpFoundation (which is used in Laravel) provides a [StreamdResponse](http://symfony.com/doc/current/components/http_foundation/introduction.html#streaming-a-response), which would be something like this:

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

$response = new StreamedResponse();
$response->setCallback(function () {
  $out = fopen('php://output', 'w');
  fputcsv($out, array('this','is some', 'csv "stuff", you know.'));
  fclose($out);
});
$response->send();
```

The CSV isn't actually outputted until `send()` is called (which is done automatically later on by Laravel) and you can add headers if you want. So let's create a basic example to output a CSV directly file directly to the browser:

```php
    use Symfony\Component\HttpFoundation\StreamedResponse;

    Route::get('export', function(){
        $response = new StreamedResponse(function(){
            // Open output stream
            $handle = fopen('php://output', 'w');
            
            // Add CSV headers
            fputcsv($handle, [
                'id',
                'name', 
                'email'
            ]);
            
            // Get all users
            foreach (User::all() as $user) {
                // Add a new row with data
                fputcsv($handle, [
                    $user->id,
                    $user->name,
                    $user->email
                ]);
            }
            
            // Close the output stream
            fclose($handle);
        }, 200, [
                'Content-Type' => 'text/csv',
                'Content-Disposition' => 'attachment; filename="export.csv"',
            ]);

        return $response;
    });
```

So this should force the download of a CSV file with your users. 

### Chunking large queries
But what if you have thousands of rows in your database? The Streaming would work, but you can fill your memory up when you load all that data. Luckily Laravel provides the [`chunk()`](http://laravel.com/docs/5.0/queries#selects) method on its Query Builder. You can provide a number of rows to fetch at once, and a callback to run on the result. So we can replace the `User::all()` with the following:

```php
    \User::chunk(500, function($users) use($handle) {
        foreach ($users as $user) {
            // Add a new row with data
            fputcsv($handle, [
              $user->id,
              $user->name,
              $user->email
            ]);
        }
    });
```

This will keep your memory within limits. 

### Some tips
 - Remember that you still have to check your execution time, so you can set it to unlimited with `set_time_limit(0);` or reset it every chunk.
 - Exporting UTF-8 data? Set the UTF-8 BOM directly after opening the stream:
  `fputs($handle, chr(0xEF) . chr(0xBB) . chr(0xBF));`
 - Check your countries settings for the delimiter. In the Netherlands for example, MS Excel recognizes the `;` delimiter, but not the (default) `,`.
 - Is it still taking too long? You could always use a Queue to create a file in the background and present it as a download when ready.
