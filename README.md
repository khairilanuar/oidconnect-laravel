![](https://habrastorage.org/web/7c1/a19/e76/7c1a19e76cf54cb1adf2217a156b7310.png)

The OpenIDConnect Laravel package is meant to provide you an opportunity to easily authenticate users using OpenID Connect protocol.

[![Latest Stable Version](https://poser.pugx.org/muffycompo/oidconnect-laravel/v/stable)](https://packagist.org/packages/muffycompo/oidconnect-laravel)
[![Latest Unstable Version](https://poser.pugx.org/muffycompo/oidconnect-laravel/v/unstable)](https://packagist.org/packages/muffycompo/oidconnect-laravel)
[![Total Downloads](https://poser.pugx.org/muffycompo/oidconnect-laravel/downloads)](https://packagist.org/packages/muffycompo/oidconnect-laravel)
[![License](https://poser.pugx.org/muffycompo/oidconnect-laravel/license)](https://packagist.org/packages/muffycompo/oidconnect-laravel)

## Installation

To install this package you will need:
* Laravel 5.5+
* PHP 7.1+

Use composer to install
```bash
composer require ivarsju/oidconnect-laravel:dev-master
```

Open `config/app.php` and register the required service providers above your application providers.
```php
'providers' => [
    ...
    Laravel\Socialite\SocialiteServiceProvider::class,
    Furdarius\OIDConnect\ServiceProvider::class
    ...
]
```

If you'd like to make configuration changes in the configuration file you can pubish it with the following Aritsan command:
```bash
php artisan vendor:publish --provider="Furdarius\OIDConnect\ServiceProvider"
```

After that, roll up migrations:
```bash
php artisan migrate
```

## Usage


#### Configuration
At first you will need to add credentials for the OpenID Connect service your application utilizes.
These credentials should be placed in your `config/opidconnect.php` configuration file.

NB! Multiple drivers are supported but this is not production ready, KeysFetcher and TokenRefresher is not properly configured, it's not important for me at the moment, but you better don't use this fork, use original repository, sorry :)

```php
<?php

return [
    'myoidc' => [
        'client_id' => '',
        'client_secret' => '',
        'redirect' => env('APP_URL') . '/auth/callback',
        'auth' => 'https://opidc.provider/auth',
        'token' => 'https://opidc.provider/token',
        'keys' => 'https://opidc.provider/keys',
        'scopes' => [],
        'guzzle' => [],
        'response_type' => 'code id_token',
    ],
];
```

#### Endpoints
Define redirect, callback and refresh routes:
```
Route::get('/auth/redirect/{driver}', '\Furdarius\OIDConnect\Http\Controllers\AuthController@redirect');
Route::get('/auth/callback/{driver}', '\Furdarius\OIDConnect\Http\Controllers\AuthController@callback');
Route::post('/auth/refresh/{driver}', '\Furdarius\OIDConnect\Http\Controllers\AuthController@refresh');
```

Now, your app has auth endpoints:
* `GET /auth/redirect` - Used to redirect client to Auth Service login page.
* `GET /auth/callback` - Used when Auth Service redirect client to callback url with code.
* `POST /auth/refresh` - Used by client for ID Token refreshing.

#### Middleware
You need to use Auth Middleware on protected routes.
Open `App\Http\Kernel` and register middleware in `$routeMiddleware`:
```php
protected $routeMiddleware = [
    'token' => \Furdarius\OIDConnect\TokenMiddleware::class
];
```

And then use it as usual:
```php
Route::middleware('token')->get('/protected-resource', function (Illuminate\Http\Request $request) {
    return "You are on protected zone";
});
```

#### User Auth

Create your own `StatelessGuard` and setup it in `config/auth.php`. Example:

Guard:
```php
<?php

namespace App\Auth;

use Illuminate\Auth\AuthenticationException;
use Illuminate\Auth\GuardHelpers;
use Illuminate\Contracts\Auth\Authenticatable;
use Illuminate\Contracts\Auth\Guard;
use Illuminate\Support\Traits\Macroable;

class StatelessGuard implements Guard
{
    use GuardHelpers, Macroable;

    /**
     * @return \Illuminate\Contracts\Auth\Authenticatable
     * @throws AuthenticationException
     */
    public function user()
    {
        if (null === $this->user) {
            throw new AuthenticationException('Unauthenticated user');
        }

        return $this->user;
    }

    /**
     * @param array $credentials
     * @return bool
     */
    public function validate(array $credentials = [])
    {
        return $this->user instanceof Authenticatable;
    }
}
```

Config (`config/auth.php`):

```php
'defaults' => [
    'guard' => 'stateless',
    'passwords' => 'users',
],

...

'guards' => [
    'stateless' => [
        'driver' => 'stateless'
    ]
],
```


Then implement own `Authenticator`. Example:

```php
<?php

namespace App\Auth;

use App\User;
use Furdarius\OIDConnect\Contract\Authenticator;
use Furdarius\OIDConnect\Exception\AuthenticationException;
use Lcobucci\JWT\Token\DataSet;

class PersonAuthenticatorAdapter implements Authenticator
{
    /**
     * @param DataSet $claims
     *
     * @return void
     */
    public function authUser(DataSet $claims)
    {
        $email = $claims->get('email');
        if (!$email) {
            throw new AuthenticationException('User\'s email not present in token');
        }

        $model = new User(['email' => $email]);

        \Auth::setUser($model);
    }
}
```

And implement auth guard service provider. Example:

```php
<?php

namespace App\Auth;

use Furdarius\OIDConnect\Contract\Authenticator;
use Illuminate\Support\ServiceProvider;

class AuthenticatorServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        \Auth::extend('stateless', function () {
            return new StatelessGuard();
        });
    }

    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton(Authenticator::class, function ($app) {
            return new PersonAuthenticatorAdapter();
        });
    }
}
```

Then register it in `config/app.php`:

```
'providers' => [
    ...
    App\Auth\AuthenticatorServiceProvider::class,
    ...
]
```

Now you can use `\Auth::user();` for getting current user information.
