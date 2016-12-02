# 路由

- [基本路由](#basic-routing)
- [路由參數](#route-parameters)
    - [必要參數](#required-parameters)
    - [選擇性參數](#parameters-optional-parameters)
- [命名路由](#named-routes)
- [路由群組](#route-groups)
    - [中介層](#route-group-middleware)
    - [命名空間](#route-group-namespaces)
    - [子網域路由](#route-group-sub-domain-routing)
    - [路由前綴](#route-group-prefixes)
- [路由模型綁定](#route-model-binding)
    - [隱式綁定](#implicit-binding)
    - [顯式綁定](#explicit-binding)
- [表單方法欺騙](#form-method-spoofing)
- [存取當前路由](#accessing-the-current-route)

<a name="basic-routing"></a>
## 基本路由

大多數基本的 Laravel 路由僅包含一個 URI 和一個 `閉包`，提供了簡潔有力的路由方法：

    Route::get('foo', function () {
        return 'Hello World';
    });

#### 預設路由檔案

所有的 Laravel 路由設定都被定義在你的路由檔案中，就放置在 `routes` 目錄下。而這些檔案會被自動載入使用。`routes/web.php` 檔案定義了所有會在網頁介面中使用的路由。像是 session 狀態和 CSRF 保護相關的路由設定會被指派給 `web` 中介層群組。 `routes/api.php` 中的路由設定則是無狀態的且指派給 `api` 中繼層群組。

For most applications, you will begin by defining routes in your `routes/web.php` file.

#### 可用的路由器方法

路由器能讓你註冊回應任何 HTTP 動詞的路由：

    Route::get($uri, $callback);
    Route::post($uri, $callback);
    Route::put($uri, $callback);
    Route::patch($uri, $callback);
    Route::delete($uri, $callback);
    Route::options($uri, $callback);

有時候你可能需要讓註冊一個路由可以應對到多個 HTTP 動詞。你可以使用 `match` 方法做到。或者甚至可以透過 `any` 方法來使用註冊路由並回應所有的 HTTP 動詞：

    Route::match(['get', 'post'], '/', function () {
        //
    });

    Route::any('foo', function () {
        //
    });

#### CSRF 保護

所有的 HTML 表單針對 `POST`、`PUT`、`DELETE` 等等路由設定都定義在 `web` 路由檔案中，且應該要包含一個 CSRF token 欄位。否則，該請求將被退回。你可以在 [CSRF 文件](/docs/{{version}}/csrf) 中讀到更多關於 CSRF 保護：

    <form method="POST" action="/profile">
        {{ csrf_field() }}
        ...
    </form>

<a name="route-parameters"></a>
## 路由參數

<a name="required-parameters"></a>
### 必要的路由參數

Of course, sometimes you will need to capture segments of the URI within your route. For example, you may need to capture a user's ID from the URL. You may do so by defining route parameters:

    Route::get('user/{id}', function ($id) {
        return 'User '.$id;
    });

You may define as many route parameters as required by your route:

    Route::get('posts/{post}/comments/{comment}', function ($postId, $commentId) {
        //
    });

Route parameters are always encased within `{}` braces and should consist of alphabetic characters. Route parameters may not contain a `-` character. Use an underscore (`_`) instead.

<a name="parameters-optional-parameters"></a>
### 選擇性的路由參數

Occasionally you may need to specify a route parameter, but make the presence of that route parameter optional. You may do so by placing a `?` mark after the parameter name. Make sure to give the route's corresponding variable a default value:

    Route::get('user/{name?}', function ($name = null) {
        return $name;
    });

    Route::get('user/{name?}', function ($name = 'John') {
        return $name;
    });

<a name="named-routes"></a>
## 命名路由

Named routes allow the convenient generation of URLs or redirects for specific routes. You may specify a name for a route by chaining the `name` method onto the route definition:

    Route::get('user/profile', function () {
        //
    })->name('profile');

You may also specify route names for controller actions:

    Route::get('user/profile', 'UserController@showProfile')->name('profile');

#### 對命名路由產生 URLs

Once you have assigned a name to a given route, you may use the route's name when generating URLs or redirects via the global `route` function:

    // Generating URLs...
    $url = route('profile');

    // Generating Redirects...
    return redirect()->route('profile');

If the named route defines parameters, you may pass the parameters as the second argument to the `route` function. The given parameters will automatically be inserted into the URL in their correct positions:

    Route::get('user/{id}/profile', function ($id) {
        //
    })->name('profile');

    $url = route('profile', ['id' => 1]);

<a name="route-groups"></a>
## 路由群組

Route groups allow you to share route attributes, such as middleware or namespaces, across a large number of routes without needing to define those attributes on each individual route. Shared attributes are specified in an array format as the first parameter to the `Route::group` method.

<a name="route-group-middleware"></a>
### 中介層

To assign middleware to all routes within a group, you may use the `middleware` key in the group attribute array. Middleware are executed in the order they are listed in the array:

    Route::group(['middleware' => 'auth'], function () {
        Route::get('/', function ()    {
            // Uses Auth Middleware
        });

        Route::get('user/profile', function () {
            // Uses Auth Middleware
        });
    });

<a name="route-group-namespaces"></a>
### 命名空間

Another common use-case for route groups is assigning the same PHP namespace to a group of controllers using the `namespace` parameter in the group array:

    Route::group(['namespace' => 'Admin'], function() {
        // Controllers Within The "App\Http\Controllers\Admin" Namespace
    });

Remember, by default, the `RouteServiceProvider` includes your route files within a namespace group, allowing you to register controller routes without specifying the full `App\Http\Controllers` namespace prefix. So, you only need to specify the portion of the namespace that comes after the base `App\Http\Controllers` namespace.

<a name="route-group-sub-domain-routing"></a>
### 子網域路由

Route groups may also be used to handle sub-domain routing. Sub-domains may be assigned route parameters just like route URIs, allowing you to capture a portion of the sub-domain for usage in your route or controller. The sub-domain may be specified using the `domain` key on the group attribute array:

    Route::group(['domain' => '{account}.myapp.com'], function () {
        Route::get('user/{id}', function ($account, $id) {
            //
        });
    });

<a name="route-group-prefixes"></a>
### 路由前綴

The `prefix` group attribute may be used to prefix each route in the group with a given URI. For example, you may want to prefix all route URIs within the group with `admin`:

    Route::group(['prefix' => 'admin'], function () {
        Route::get('users', function ()    {
            // Matches The "/admin/users" URL
        });
    });

<a name="route-model-binding"></a>
## 路由模型綁定

When injecting a model ID to a route or controller action, you will often query to retrieve the model that corresponds to that ID. Laravel route model binding provides a convenient way to automatically inject the model instances directly into your routes. For example, instead of injecting a user's ID, you can inject the entire `User` model instance that matches the given ID.

<a name="implicit-binding"></a>
### 隱式綁定

Laravel automatically resolves Eloquent models defined in routes or controller actions whose variable names match a route segment name. For example:

    Route::get('api/users/{user}', function (App\User $user) {
        return $user->email;
    });

In this example, since the Eloquent `$user` variable defined on the route matches the `{user}` segment in the route's URI, Laravel will automatically inject the model instance that has an ID matching the corresponding value from the request URI. If a matching model instance is not found in the database, a 404 HTTP response will automatically be generated.

#### 自定鍵名

If you would like model binding to use a database column other than `id` when retrieving a given model class, you may override the `getRouteKeyName` method on the Eloquent model:

    /**
     * Get the route key for the model.
     *
     * @return string
     */
    public function getRouteKeyName()
    {
        return 'slug';
    }

<a name="explicit-binding"></a>
### 顯式綁定

To register an explicit binding, use the router's `model` method to specify the class for a given parameter. You should define your explicit model bindings in the `boot` method of the `RouteServiceProvider` class:

    public function boot()
    {
        parent::boot();

        Route::model('user', App\User::class);
    }

Next, define a route that contains a `{user}` parameter:

    $router->get('profile/{user}', function(App\User $user) {
        //
    });

Since we have bound all `{user}` parameters to the `App\User` model, a `User` instance will be injected into the route. So, for example, a request to `profile/1` will inject the `User` instance from the database which has an ID of `1`.

If a matching model instance is not found in the database, a 404 HTTP response will be automatically generated.

#### 自定解析邏輯

If you wish to use your own resolution logic, you may use the `Route::bind` method. The `Closure` you pass to the `bind` method will receive the value of the URI segment and should return the instance of the class that should be injected into the route:

    $router->bind('user', function ($value) {
        return App\User::where('name', $value)->first();
    });

<a name="form-method-spoofing"></a>
## 表單方法欺騙

HTML forms do not support `PUT`, `PATCH` or `DELETE` actions. So, when defining `PUT`, `PATCH` or `DELETE` routes that are called from an HTML form, you will need to add a hidden `_method` field to the form. The value sent with the `_method` field will be used as the HTTP request method:

    <form action="/foo/bar" method="POST">
        <input type="hidden" name="_method" value="PUT">
        <input type="hidden" name="_token" value="{{ csrf_token() }}">
    </form>

You may use the `method_field` helper to generate the `_method` input:

    {{ method_field('PUT') }}

<a name="accessing-the-current-route"></a>
## 存取當前路由

You may use the `current`, `currentRouteName`, and `currentRouteAction` methods on the `Route` facade to access information about the route handling the incoming request:

    $route = Route::current();

    $name = Route::currentRouteName();

    $action = Route::currentRouteAction();

Refer to the API documentation for both the [underlying class of the Route facade](http://laravel.com/api/{{version}}/Illuminate/Routing/Router.html) and [Route instance](http://laravel.com/api/{{version}}/Illuminate/Routing/Route.html) to review all accessible methods.
