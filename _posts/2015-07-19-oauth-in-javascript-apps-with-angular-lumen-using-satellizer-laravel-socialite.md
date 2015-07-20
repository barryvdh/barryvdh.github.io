---
layout:     post
title:      OAuth in Javascript Apps with Angular and Lumen, using Satellizer and Laravel Socialite
date:       2015-07-19 22:00:00
excerpt:    There a lot of blogs about OAuth an Laravel Socialite, but in this post I explain how to use this in your Angular app + Laravel/Lumen API.
categories: laravel lumen angular
---

In the last few weeks, Socialite was a popular topic to blog/tweet about. Coincidentally, I also needed Socialite for a project. But in my case, I wanted to use it in an Angular app, distributed using Cordova (Phonegap) as hybrid app on Android/iOS. There were some examples, but I couldn't find much about it at the time. A few people asked to share my experience about it, so here it is!

## Before we start

As I said, a lot has been written about Socialite. So I'm going to assume you have Socialite working with Laravel or Lumen already. If you don't, just read these links:

 - [Laravel Docs - Social Authentication](http://laravel.com/docs/5.1/authentication#social-authentication)
 - [Using Github Authentication for login with Laravel Socalite by Matt Stauffer](https://mattstauffer.co/blog/using-github-authentication-for-login-with-laravel-socialite)
 
And off course some basis Angular knowledge would come in handy when you're building an Angular app. So we're also not covering installing Satellizer, but you can grab [the example from the Github repo here](https://github.com/sahat/satellizer/tree/master/examples/client).

## Tools

So what are we going to use exactly?

 - [Lumen](http://lumen.laravel.com/) a.k.a. Laravel Light, because we're just building an API and want more speed. (But everything in this blog applies for Laravel also).
 - [Angular](https://angularjs.org/), the client-side framework we're using.
 - [Satellizer](https://github.com/sahat/satellizer), the Angular library for OAuth + Token based authentication.
 - [Socialite](https://github.com/laravel/socialite), the 'official' library for OAuth in Laravel.
 
## The Flow
 
So what are we hoping to achieve? In my case:
 
 - Should work on every device, so on a regular domain, but also on a hybrid Android/iOS app (using Cordova)
 - Use both OAuth 1 (Twitter) and OAuth 2 (Facebook/LinkedIn etc)
 - Get profile information from Socialite.
 - Authenticate users using JWT (JSON Web Tokens)
 
Luckily Satellizer provides us some info about how this should work with [OAuth2](https://github.com/sahat/satellizer/wiki/Login-with-OAuth-2.0):

1. **Client:** Open a popup window via `$auth.authenticate('provider name')`.
2. **Client:** Sign in with that provider, if necessary, then authorize the application.
3. **Client:** After successful authorization, the popup is redirected back to 
your app, e.g. *http://localhost:3000*,  with the `code` (authorization code)
query string parameter.
4. **Client:** The `code` parameter is sent back to the  parent window that opened the popup.
5. **Client:** Parent window closes the popup and sends a **POST** 
request to */auth/provider* with`code` parameter.
6. **Server:** *Authorization code* is exchanged for *access token*.
7. **Server:** User information is retrieved using the *access token* from **Step 6**.
8. **Server:** Look up the user by their unique *Provider ID*. If user already 
exists, grab the existing user, otherwise create a new user account.
9. **Server:** In both cases of Step 8, create a JSON Web Token and send it back to the client.
10. **Client:** Parse the token and save it to *Local Storage* for subsequent
use after page reload.

[OAuth1](https://github.com/sahat/satellizer/wiki/Login-with-OAuth-1.0) has some more steps, but you get the idea.

It sounds a bit complex (well, at least that was what I thought the first time I read it..), but let's translate it for our use-case.

1. Call `$auth.authenticate('google').then(successCallback)` in your **Angular app** and the popup opens.
2. The **end-user** logs in using his Social account.
3. The **Social provider** redirects to the URL you choose. The url should be the same as the domain your are on, otherwise you can't access the code parameter. So for Cordova apps, `http://localhost:3000` is fine. For apps on a 'real' domain, you can just use that. Just make sure it's an allowed url. So you _don't_ use the Laravel API url as redirectUri!
4. **Satellizer** reads the code.
5. A POST request with the Authorization code is sent to the **Lumen API**.
6. **Socialite** exchanges the code for an access token.
7. **Socialite** uses this token to get the profile data.
8. **Lumen** either creates a new User or looks up the existing one (using the unique id of the profile)
9. A JSON Web Token is returned to **Satellizer**.
10. **Satellizer** stores the token for later use.

This is actually much easier than it looks, because Satellizer and Socialite do all the heavy lifting. 

## Handling the OAuth calls in Lumen

So how do we use Socialite to get the profile? Satellizer already has [an example using Laravel](https://github.com/sahat/satellizer/tree/master/examples/server/php) but it doesn't use Socialite, so it's different for each provider. For example, Github:

```php
 /**
 * Login with GitHub.
 */
public function github(Request $request)
{
    $accessTokenUrl = 'https://github.com/login/oauth/access_token';
    $userApiUrl = 'https://api.github.com/user';
    
    $params = [
        'code' => $request->input('code'),
        'client_id' => $request->input('clientId'),
        'client_secret' => Config::get('app.github_secret'),
        'redirect_uri' => $request->input('redirectUri')
    ];
    
    $client = new GuzzleHttp\Client();
    
    // Step 1. Exchange authorization code for access token.
    $accessTokenResponse = $client->get($accessTokenUrl, ['query' => $params]);
    $accessToken = array();
    parse_str($accessTokenResponse->getBody(), $accessToken);
    $headers = array('User-Agent' => 'Satellizer');
    
    // Step 2. Retrieve profile information about the current user.
    $userApiResponse = $client->get($userApiUrl, [
        'headers' => $headers,
        'query' => $accessToken
    ]);
    $profile = $userApiResponse->json();
    
    // Step 3a. If user is already signed in then link accounts.
    if ($request->header('Authorization'))
    {
        $user = User::where('github', '=', $profile['id']);
        if ($user->first())
        {
            return response()->json(['message' => 'There is already a GitHub account that belongs to you'], 409);
        }
        
        $token = explode(' ', $request->header('Authorization'))[1];
        $payload = (array) JWT::decode($token, Config::get('app.token_secret'), array('HS256'));
        
        $user = User::find($payload['sub']);
        $user->github = $profile['id'];
        $user->displayName = $user->displayName || $profile['name'];
        $user->save();
        
        return response()->json(['token' => $this->createToken($user)]);
    }
    // Step 3b. Create a new user account or return an existing one.
    else
    {
        $user = User::where('github', '=', $profile['id']);
        if ($user->first())
        {
            return response()->json(['token' => $this->createToken($user->first())]);
        }
        
        $user = new User;
        $user->github = $profile['id'];
        $user->displayName = $profile['name'];
        $user->save();
        
        return response()->json(['token' => $this->createToken($user)]);
    }
}
```

So yeah, not very pretty when we have many providers. But we can see what it does:

- Step 1: Exchange the received Authorization code for an Access token
- Step 2: Use token to get the Social profile.
- Step 3a: When already logged in, connect this network to the logged in user.
- Step 3b: Otherwise look up the user with this profile id, or create a new one.
- Return a token for Satellizer.

How can we do this with Socialite? The first 2 steps can be done by Socialite directly. We just need to set the correct redirectUri, as provided by the Request from Satellizer. We also setting it to be stateless, because we're not using the Session for the API.

```php
    if ($request->has('redirectUri')) {
        config()->set("services.{$name}.redirect", $request->get('redirectUri'));
    }

    $provider = Socialite::driver($name);
    $provider->stateless();
    
    // Step 1 + 2
    $profile = $provider->user();
    
    // Handle the user etc.
```

The third step is pretty much the same, but we can use the standardized `$profile->getId()`, `$profile->getName()`, `$profile->getEmail` etc.

For OAuth1, the flow is a bit different. And I must say I'm not really happy with this, but just going to leave it at this anyway.

```php
// Part 1 of 2: Initial request from Satellizer.
if ( ! $request->input('oauth_token') || ! $request->input('oauth_verifier')) {
    // Redirect to fill the session (without actually redirecting)
    $provider->redirect();

    /** @var TemporaryCredentials $temp */
    $credentials = $request->getSession()->get('oauth.temp');

    return response()->json(['oauth_token' => $credentials->getIdentifier()]);
}
// Part 2 of 2: Second request after Authorize app is clicked.
else
{
    $credentials = new TemporaryCredentials();
    $credentials->setIdentifier($request->input('oauth_token'));
    $request->getSession()->set('oauth.temp', $credentials);

    // Step 1 + 2
    $profile = $provider->user();
    
    // Handle the user etc.
}
```

So we're kind of faking the Session, because A) we're not using the session and B) We can't use the session because Satellizer makes the actual request. This could probably be improved somehow, but would require some changes in Satellizer.

## The JWT tokens

So, as you see in the example, we're calling `$this->createToken($user)` to create the token. We're just going to use the ['regular' php-jwt package from Firebase](https://github.com/firebase/php-jwt). If you want to use [a Laravel-specific package](https://github.com/tymondesigns/jwt-auth), read [this blog from Ryan Chenkie](https://scotch.io/tutorials/token-based-authentication-for-angularjs-and-laravel-apps)

It's actually pretty simple, something like this:

```php
/**
 * Generate JSON Web Token.
 * @param  User  $user
 * @return string
 */
protected function createToken($user)
{
    $payload = [
      'sub' => $user->getAuthIdentifier(),
      'iat' => time(),
      'exp' => time() + (365 * 24 * 60 * 60), // 1 year
    ];

    return \JWT::encode($payload, env('JWT_KEY'), 'HS256');
}
```

Satellizer can parse the token to find out if it's still valid (you can't invalidate it, it just expires). Unfortunately there isn't an option to refresh a token yet in Satellizer, but you could probably build it yourself. Otherwise the users just gets logged out after a year.

Satellizer sends the token in the Authorization header, so we can verify this. We'll create an middleware for this. We're doing it a bit different than [the example](https://github.com/sahat/satellizer/blob/8b8ca04f6de3278205edfbc763df2d86a3f9254c/examples/server/php/app/Http/Middleware/Authenticate.php) (again):

```php
public function handle($request, Closure $next)
{
    if ($request->header('Authorization')) {
        $token = explode(' ', $request->header('Authorization'))[1];

        $payload = (array) JWT::decode($token, env('JWT_KEY'), ['HS256']);

        if ($payload['exp'] < time()) {
            return response()->json(['message' => 'Token has expired']);
        }

        Auth::onceUsingId($payload['sub']);
    }

    return $next($request);
}
```

So we're logging this user in for just his one request (we're not using a session, remember). That means we can use `Auth::check()` and `Auth::user()` in our code. No need for different JWT/Auth libraries, just our checks as usual. You can still combine this middleware with the normal `auth` middleware.

So this is pretty much all that's needed to get your Angular app to play nice with Lumen (or Laravel, which is pretty much exactly the same). If you have any improvements or tips, feel free to open an issue/pull request on [barryvdh/barryvdh.github.io](https://github.com/barryvdh/barryvdh.github.io)

## Some gotcha's 

- You need to add some lines to you `.htaccess` to actually get the Authorization Header in your Laravel code:

```
# Include Authorization header in a request
RewriteCond %{HTTP:Authorization} ^(.+)$
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

- If you're using a different domain, you'll need to configure your CORS headers. Try https://github.com/barryvdh/laravel-cors
 
 
