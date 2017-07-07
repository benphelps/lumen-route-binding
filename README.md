# Lumen Route Binding

[![Build Status](https://travis-ci.org/mmghv/lumen-route-binding.svg?branch=master)](https://travis-ci.org/mmghv/lumen-route-binding)
[![Lumen Version](https://img.shields.io/badge/Lumen-5.0%20to%205.4-orange.svg)](https://github.com/laravel/lumen) 
[![Latest Stable Version](https://poser.pugx.org/mmghv/lumen-route-binding/v/stable)](https://packagist.org/packages/mmghv/lumen-route-binding)
[![Total Downloads](https://poser.pugx.org/mmghv/lumen-route-binding/downloads)](https://packagist.org/packages/mmghv/lumen-route-binding)
[![Latest Unstable Version](https://poser.pugx.org/mmghv/lumen-route-binding/v/unstable)](https://packagist.org/packages/mmghv/lumen-route-binding)
[![License](https://poser.pugx.org/mmghv/lumen-route-binding/license)](LICENSE)

This package Adds support for `Route Model Binding` in Lumen (5.0 to 5.4).

> As known, Lumen doesn't support `Route Model Binding` out of the box due to the fact that Lumen doesn't use the Illuminate router that Laravel uses, Instead, It uses the [FastRoute](https://github.com/nikic/FastRoute) which is much faster. With this package, We add support for the powerful `Route Model Binding` while still benefit the speed of the FastRoute in Lumen.

# Table of Contents

  * [Installation](#installation)
  * [Defining the binding](#where-to-define-our-bindings)
    * [Explicit Binding](#1-explicit-binding)
    * [Implicit Binding](#2-implicit-binding)
    * [Composite Binding](#3-composite-binding)
  * [DingoAPI Integration](#dingoapi-Integration)
  
## Installation

#### Using composer
```
composer require mmghv/lumen-route-binding "^1.0"
```

> Requires
> ```
> php >= 5.4.0
> Lumen 5.*
> ```

#### Register the service provider

You have 2 options, continue reading ..


## Usage

> Route model binding provides a convenient way to automatically inject the model instances directly into your routes. For example, instead of injecting a user's ID, you can inject the entire `User` model instance that matches the given ID.

### Where to Define our Bindings

As mentioned before, You have 2 options :

#### OPTION 1

To register the service provider included with the package and define your bindings in `bootstrap/app.php` :

```PHP
// Register the package's service provider
$app->register('mmghv\LumenRouteBinding\RouteBindingServiceProvider');

// Get the $binder instance from the IoC Container
$binder = $app['bindingResolver'];

// Then define your bindings here ..
```

#### OPTION 2 (better)

To create a dedicated service provider that extendes the package's one and place it in `app/Providers` :

```PHP
// app/Providers/RouteBindingServiceProvider.php

namespace App\Providers;

use mmghv\LumenRouteBinding\RouteBindingServiceProvider as BaseServiceProvider;

class RouteBindingServiceProvider extends BaseServiceProvider
{
    /**
     * Boot the service provider
     */
    public function boot()
    {
        // The binder instance
        $binder = $this->binder;

        // Here we define our bindings
    }
}
```

Then register this provider rather than the package's one in `bootstrap/app.php` :

```PHP
$app->register('App\Providers\RouteBindingServiceProvider');
```

This way, We can define our `bindings` in a dedicated service provider (in the `boot` method).

### Defining the Bindings

We have **Three** types of bindings:

#### 1) Explicit Binding

We can explicitly bind a route wildcard name to a specific model using the `bind` method :

```PHP
$binder->bind('user', 'App\User');
```

This way, Anywhere in our routes if the wildcard `{user}` is found, It will be resolved to the `User` model instance that corresponds to the wildcard value, So we can define our route like this :

```PHP
$app->get('profile/{user}', function(App\User $user) {
    //
});
```

Behind the scenes, The binder will resolve the model instance like this :

```PHP
$instance = new App\User;
return $instance->where($instance->getRouteKeyName(), $value)->firstOrFail();
```

##### Customizing The Key Name

By default, It will use the model's ID column. Like Laravel, If you would like it to use another column when retrieving a given model class, you may override the `getRouteKeyName` method on the Eloquent model :

```PHP
/**
 * Get the route key for the model.
 *
 * @return string
 */
public function getRouteKeyName()
{
    return 'slug';
}
```

##### Using a Custom Resolver Callback :

If you wish to use your own resolution logic, you may pass a `Class@method` callable style or a `Closure` instead of the class name to the `bind` method, The callable will receive the value of the URI segment and should return the instance of the class that should be injected into the route :

```PHP
// Using a 'Class@method' callable style
$binder->bind('article', 'App\Article@findForRoute');

// Using a closure
$binder->bind('article', function($value) {
    return \App\Article::where('slug', $value)->firstOrFail();
});
```

##### Handling the `NotFound` Exception :

If no model found with the given key, The Eloquent `firstOrFail` will throw a `ModelNotFoundException`, To handle this exception, We can pass a closure as the third parameter to the `bind` method :

```PHP
$binder->bind('article', 'App\Article', function($e) {
    // We can return a default value if the model not found :
    return new \App\Article();

    // Or we can throw another exception for example :
    throw new \NotFoundHttpException;
});
```

#### 2) Implicit Binding

Using the `implicitBind` method, We can tell the binder to automatically bind all the models in a given namespace :

```PHP
$binder->implicitBind('App');
```

So in this example :

```PHP
$app->get('articles/{article}', function($myArticle) {
    //
});
```

The binder will first check for any **explicit binding** that matches the `article` key. If no match found, It then (and according to our previous implicit binding) will check if the following class exists `App\Article` (The namespace + ucFirst(the key)), If it finds it, Then it will call `firstOrFail` on the class like the explicit binding and inject the returned instance into the route, **However**, If no classes found with this name, It will continue to the next binding (if any) and return the route parameters unchanged if no bindings matches.

##### Customizing The Key Name

Similar to binding, we could specify another column to be used to retrieve the model instance by overriding the `getRouteKeyName` method on the Eloquent model :

```PHP
/**
 * Get the route key for the model.
 *
 * @return string
 */
public function getRouteKeyName()
{
    return 'slug';
}
```

##### Implicit Binding with Repositories

We can use implicit binding with classes other than the `Eloquent` models, For example if we use something like `Repository Pattern` and would like our bindings to use the repository classes instead of the Eloquent models, We can do that.

The problem is that the repository classes names usually use a `Prefix` and\or `Suffix` beside the Eloquent model name, For example, The `Article` Eloquent model, May have a corresponding repository class with the name `EloquentArticleRepository`, We can set our implicit binding to use this prefix and\or suffix like this :

```PHP
$binder->implicitBind('App\Repositories', 'Eloquent', 'Repository');
```

(Of course we can leave out the `prefix` and\or the `suffix` if we don't use it)

So in this example :

```PHP
$app->get('articles/{article}', function($myArticle) {
    //
});
```

The binder will check if the following class exists `App\Repositories\EloquentArticleRepository` (The namespace + prefix + ucFirst(the key) + suffix), If it finds it, Then it will call `firstOrFail()` using the column from `getRouteKeyName()` (so you should have these methods on your repository).

##### Using Custom Method

If you want to use a custom method on your class to retrieve the model instance, You can pass the method name as the fourth parameter :

```PHP
$binder->implicitBind('App\Repositories', 'Eloquent', 'Repository', 'findForRoute');
```

This way, The binder will call the custom method `findForRoute` on our repository passing the route wildcard value and expecting it to return the resolved instance.

###### Example of using a custom method with Implicit Binding while using the Repository Pattern :

1- defining our binding in the service provider :

```PHP
$binder->implicitBind('App\Repositories', '', 'Repository', 'findForRoute');
```

2- defining our route in `routes.php` :

```PHP
$app->get('articles/{article}', function(App\Article $article) {
    return view('articles.view', compact('article'));
});
```

3- Adding our custom method in our repository in `apps/Repositories/ArticleRepository.php` :

```PHP
/**
 * Find the Article for route-model-bining
 *
 * @param  string $val  wildcard value
 *
 * @return \App\Article
 */
public function findForRoute($val)
{
    return $this->model->where('slug', $val)->firstOrFail();
}
```

##### Handling the `NotFound` Exception :

Similar to binding, We can handle the exception thrown in the resolver method (the model `firstOrFail` or in our repository) by passing a closure as the fifth parameter to the method `implicitBind` :

```PHP
$binder->implicitBind('App\Repositories', '', 'Repository', 'findForRoute', function($e) {
    // Do something with the exception
});
```

#### 3) Composite Binding

Sometimes, you will have a route of two or more levels that contains wildcards of related models, Something like :

```PHP
$app->get('posts/{post}/comments/{comment}', function(App\Post $post, App\Comment $comment) {
    //
});
```

In this example, If we use explicit or implicit binding, Each model will be resolved individually with no relation to each other, Sometimes that's OK, But what if we want to resolve these models in one binding to handle the relationship between them and maybe do a proper eager loading without repeating the process for each model individually, That's where `Composite Binding` comes into play.

In `Composite Binding` we tell the binder to register a binding for multiple wildcards in a specific order.

We use the method `compositeBind` passing an array of wildcards names as the first parameter, and a resolver callback (either a closure or a `Class@method` callable style) as the second parameter.

```PHP
// Using a 'Class@method' callable style
$binder->compositeBind(['post', 'comment'], 'App\Repositories\PostRepository@findPostCommentForRoute');

// Using a closure
$binder->compositeBind(['post', 'comment'], function($postKey, $commentKey) {
    $post = \App\Post::findOrFail($postKey);
    $comment = $post->comments()->findOrFail($commentKey);

    return [$post, $comment];
});
```

**Note:**
This binding will match the route that has **only** and **exactly** the given wildcards (in this case `{post}` and `{comment}`) and they appear in the same exact **order**. The resolver callback will be handled the wildcards values and **MUST** return the resolved models in an array of the same count and order of the wildcards.

**Note:**
This type of binding takes a priority over any other type of binding, Meaning that in the previous example if we have an explicit or implicit binding for `post` and\or `comment`, None of them will take place as long as the route **as whole** matches a composite binding.

##### Handling the `NotFound` Exception :

Similar to and implicit binding, We can handle the exception thrown in the resolver callback by passing a closure as the third parameter to the method `compositeBind` :

```PHP
$binder->compositeBind(['department', 'section'], 'App\Department@getDepartmentAndSection', function($e) {
    // Do something with the exception
});
```

## DingoAPI Integration

If you're using [DingoAPI](https://github.com/dingo/api) and want to use `LumenRouteBinding` with it, You will need to extend `DingoAPI` to do that .. so you can do the following :

#### 1- Extent LumenAdapter class

Create the following class named `DingoAdapter` that will extent the class `Dingo\Api\Routing\Adapter\Lumen` and put it somewhere, lets say in `app\Custom` :

```PHP
// app/Custom/DingoAdapter.php

namespace App\Custom;

use Illuminate\Http\Request;
use Dingo\Api\Exception\UnknownVersionException;
use Dingo\Api\Routing\Adapter\Lumen as BaseDingoAdapter;

class DingoAdapter extends BaseDingoAdapter
{
    /**
     * Dispatch a request.
     *
     * @param \Illuminate\Http\Request $request
     * @param string                   $version
     *
     * @return mixed
     */
    public function dispatch(Request $request, $version)
    {
        if (! isset($this->routes[$version])) {
            throw new UnknownVersionException;
        }

        $this->removeMiddlewareFromApp();

        $routes = $this->routes[$version];

        // This is what we extended the class for, to use the dispatcher created
        // by LumenRouteBinding instead of creating a new one.
        $this->app['dispatcher']->setRoutesResolver(function() use ($routes) {
            return $routes->getData();
        });

        $this->normalizeRequestUri($request);

        return $this->app->dispatch($request);
    }
}
```

#### 2- Extent Dingo ServiceProvider

Next we need to extend the service provider to use the custom class, Create the following class in `app/Providers` :

```PHP
// app/Providers/DingoServiceProvider.php

namespace App\Providers;

use App\Custom\DingoAdapter;
use FastRoute\RouteParser\Std as StdRouteParser;
use FastRoute\DataGenerator\GroupCountBased as GcbDataGenerator;
use Dingo\Api\Provider\LumenServiceProvider as BaseDingoServiceProvider;

class DingoServiceProvider extends BaseDingoServiceProvider
{
    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        parent::register();

        // Replace dingo adapter with an extended one to use the dispatcher from LumenRouteBinding
        $this->app->singleton('api.router.adapter', function ($app) {
            return new DingoAdapter($app, new StdRouteParser, new GcbDataGenerator, '');
        });
    }
}
```

#### 3- Register the custom service provider

Then in `bootstrap/app.php` replace the registeration of `Dingo` service provider with the custom one:

```PHP
// remove this line
$app->register(Dingo\Api\Provider\LumenServiceProvider::class);

// and add this line
$app->register(App\Providers\DingoServiceProvider::class);
```

And don't forget to register the LumenRoutebinding service provider itself 

```PHP
$app->register(App\Providers\RouteBindingServiceProvider::class);
```

That's it, Now you shoud be able to use `LumenRoutebinding` with `DingoAPI`.

## Contributing
If you found an issue, Please report it [here](https://github.com/mmghv/lumen-route-binding/issues).

Pull Requests are welcome, just make sure to follow the PSR-2 standards and don't forget to add tests.

## License & Copyright

Copyright © 2016, [Mohamed Gharib](https://github.com/mmghv).
Released under the [MIT license](LICENSE).