Laravel Static Html Cache
=========================

PHP is awesome and (HHVM)[http://hhvm.com/] makes it faster than ever. However it will still suffer under extreme bursts of traffic. Static site generators are a great way to get around this, but they are only sutible for a small subset of situations. Here are instructions to create static html files from your laravel application and serve them directly and improve load time and lower CPU usage.

You can easily cache pages by adding the cache.html filter to the route.

I might add some artisan commands to clear the cache, but at this point I think it's sufficient that it is rebuild every hour.

## How To
1. Setup your .htaccess
2. Add the filter

### Setup your .htaccess

```
# Rewrite to html cache if it exists and the request is off a static page
# (no url query params, no session cookies and only get requests)
# A new cache will be created every hour
RewriteCond %{REQUEST_METHOD} GET
RewriteCond %{QUERY_STRING} !.*=.*
RewriteCond %{HTTP_COOKIE} !^.*(cartalyst_sentry).*$
RewriteCond %{DOCUMENT_ROOT}/cache/html/%{TIME_DAY}/%{TIME_HOUR}/%{HTTP_HOST}/$1/index.html -f
RewriteRule ^(.*) /cache/html/%{TIME_DAY}/%{TIME_HOUR}/%{HTTP_HOST}/$1/index.html [L]
```

or for Ngix (output from http://www.anilcetin.com/convert-apache-htaccess-to-nginx/)

```
if ($request_method ~ "GET") {
	set $rule_0 1$rule_0;
}
if ($args !~ ".*=.*") {
	set $rule_0 2$rule_0;
}
if ($http_cookie !~ "^.*(cartalyst_sentry).*$") {
	set $rule_0 3$rule_0;
}
if (-f $document_root/cache/html/%{TIME_DAY}/%{TIME_HOUR}/$http_host/$1/index.html) {
	set $rule_0 4$rule_0;
}
if ($rule_0 = "4321") {
	rewrite ^/(.*) /cache/html/%{TIME_DAY}/%{TIME_HOUR}/$http_host/$1/index.html last;
}
```

### Add the filter

Add this to your app/filters.php

```php
/*
|--------------------------------------------------------------------------
| HTML Caching Filter
|--------------------------------------------------------------------------
|
| The HTML Caching filter is responsible for caching the html output of
| your views. These pages can then be served statically as HTML by setting
| up redirects.
|
*/

Route::filter('cache.html', function($route, $request, $response = null)
{
	if ($request->method() == 'GET' and Auth::guest() and ! Config::get('app.debug'))
	{
		$day = str_pad(Date::now()->day, 2, '0', STR_PAD_LEFT);
		$hour = str_pad(Date::now()->hour, 2, '0', STR_PAD_LEFT);
		$url = preg_replace('/https?:\/\//', '', $request->url());

		$directory = public_path().'/cache/html/'.$day.'/'.$hour.'/'.$url;

		if ( ! File::isDirectory($directory))
		{
			File::makeDirectory($directory, 0777, true);
		}

		File::put($directory.'/index.html', $response->getContent());
	}
});
```

For every route you would like to be cached as static html add the filter like so

```php
Route::group(['after' => 'cache.html'], function()
{
		Route::controller('/', 'HomeController', [
			'getIndex' => 'home',
			'getEvents' => 'events',
			'getHelp' => 'help',
			'getAbout' => 'about',
			'getPrivacy' => 'privacy',
			'getTerms' => 'terms',
		]);
});
```
