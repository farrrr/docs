# 控制器

- [簡介](#introduction)
- [基礎控制器](#basic-controllers)
    - [定義控制器](#defining-controllers)
    - [控制器與命名空間](#controllers-and-namespaces)
    - [單一動作控制器](#single-action-controllers)
- [控制器中介層](#controller-middleware)
- [資源控制器](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [命名資源路由](#restful-naming-resource-routes)
    - [命名資源路由參數](#restful-naming-resource-route-parameters)
    - [附加資源控制器](#restful-supplementing-resource-controllers)
- [依賴注入與控制器](#dependency-injection-and-controllers)
- [路由快取](#route-caching)

<a name="introduction"></a>
## 簡介

除了將所有的請求處理邏輯以閉包的形式定義在路由檔案中，你或許希望能透過控制器類別的方式來組織這些邏輯行為。控制器可將相關的請求處理邏輯組成一個類別。就存放在 `app/Http/Controllers` 目錄中。

<a name="basic-controllers"></a>
## 基礎控制器

<a name="defining-controllers"></a>
### 定義控制器

以下是一個基礎控制器類別的範例。注意，控制器擴充了 Laravel 所包含的基礎控制器類別。基礎類別提供了一些方便的方法，像是 `中介層(middleware)`，可用於將中介層附加到控制器動作中：

    <?php

    namespace App\Http\Controllers;

    use App\User;
    use App\Http\Controllers\Controller;

    class UserController extends Controller
    {
        /**
         * 顯示指定的使用者個人資訊。
         *
         * @param  int  $id
         * @return Response
         */
        public function show($id)
        {
            return view('user.profile', ['user' => User::findOrFail($id)]);
        }
    }

你可以像這樣定義一個對應到此控制器動作的路由：

    Route::get('user/{id}', 'UserController@show');

現在，當請求配對到指定的路由 URI 時，`UserController` 類別的 `show` 方法將被執行。當然，路由參數也會傳遞給方法。

> {提示} 控制器並非 **一定** 得繼承基礎類別。 但是，你將無法存取一些方便的功能，像是 `中介層(middleware)`、`驗證(validate)` 以及 `任務派發(dispatch)` 之類。

<a name="controllers-and-namespaces"></a>
### 控制器與命名空間

有一點非常重要，那就是我們在定義控制器路由時，不需要指定完整的控制器命名空間。當 `RouteServiceProvider` 載入路由檔案時，其中有包含了命名空間的路由群組，所以只需要指定命名空間 `App\Http\Controllers` 之後部分的類別名稱即可。

如果你選擇在 `App\Http\Controllers` 內層使用巢狀目錄來組織控制器，只需使用相對於 `App\Http\Controllers` 根命名空間的指定類別名稱。所以，若你的完整控制器類別是 `App\Http\Controllers\Photos\AdminController`，你應該如此將路由註冊到控制器：

    Route::get('foo', 'Photos\AdminController@method');

<a name="single-action-controllers"></a>
### 單一動作控制器

如果你想定義一個只處理單個動作的控制器，你可以在控制器中放置一個  `__invoke` 方法：

    <?php

    namespace App\Http\Controllers;

    use App\User;
    use App\Http\Controllers\Controller;

    class ShowProfile extends Controller
    {
        /**
         * 顯示指定的使用者個人資訊。
         *
         * @param  int  $id
         * @return Response
         */
        public function __invoke($id)
        {
            return view('user.profile', ['user' => User::findOrFail($id)]);
        }
    }

註冊單一動作控制器的路由時，你不需要指定方法：

    Route::get('user/{id}', 'ShowProfile');

<a name="controller-middleware"></a>
## 控制器中介層

[中介層(Middleware)](/docs/{{version}}/middleware) 可以分配給路由文件中控制器的路由：

    Route::get('profile', 'UserController@show')->middleware('auth');

但是，在控制器的建構子函式中指定中介層更為方便。使用控制器建構子函式中的 `middleware` 方法，你可以輕鬆的就將中介層分配給控制器中的動作。甚至還可以將中介層限制在控制器類別中的特定方法：

    class UserController extends Controller
    {
        /**
         * 實體化新的控制器實體。
         *
         * @return void
         */
        public function __construct()
        {
            $this->middleware('auth');

            $this->middleware('log')->only('index');

            $this->middleware('subscribed')->except('store');
        }
    }

控制器還允許你使用閉包(Closure)來註冊中介層。這提供了一個簡便方法來為單一控制器定義中介層，而不需要定義整個中介層類別：

    $this->middleware(function ($request, $next) {
        // ...

        return $next($request);
    });

> {提示} 你可以將中介層分配給控制器動作的子集；不過，這可能會導致你的控制器變得太肥。相對來說，可以考慮將你的控制器打散為多個較小的控制器。

<a name="resource-controllers"></a>
## 資源控制器

Laravel resource routing assigns the typical "CRUD" routes to a controller with a single line of code. For example, you may wish to create a controller that handles all HTTP requests for "photos" stored by your application. Using the `make:controller` Artisan command, we can quickly create such a controller:

    php artisan make:controller PhotoController --resource

This command will generate a controller at `app/Http/Controllers/PhotoController.php`. The controller will contain a method for each of the available resource operations.

Next, you may register a resourceful route to the controller:

    Route::resource('photos', 'PhotoController');

This single route declaration creates multiple routes to handle a variety of actions on the resource. The generated controller will already have methods stubbed for each of these actions, including notes informing you of the HTTP verbs and URIs they handle.

#### 由資源控制器處理的行為

Verb      | URI                  | Action       | Route Name
----------|-----------------------|--------------|---------------------
GET       | `/photos`              | index        | photos.index
GET       | `/photos/create`       | create       | photos.create
POST      | `/photos`              | store        | photos.store
GET       | `/photos/{photo}`      | show         | photos.show
GET       | `/photos/{photo}/edit` | edit         | photos.edit
PUT/PATCH | `/photos/{photo}`      | update       | photos.update
DELETE    | `/photos/{photo}`      | destroy      | photos.destroy

#### 偽造表單方法

Since HTML forms can't make `PUT`, `PATCH`, or `DELETE` requests, you will need to add a hidden `_method` field to spoof these HTTP verbs. The `method_field` helper can create this field for you:

    {{ method_field('PUT') }}

<a name="restful-partial-resource-routes"></a>
### 部分資源路由

When declaring a resource route, you may specify a subset of actions the controller should handle instead of the full set of default actions:

    Route::resource('photo', 'PhotoController', ['only' => [
        'index', 'show'
    ]]);

    Route::resource('photo', 'PhotoController', ['except' => [
        'create', 'store', 'update', 'destroy'
    ]]);

<a name="restful-naming-resource-routes"></a>
### 命名資源路由

By default, all resource controller actions have a route name; however, you can override these names by passing a `names` array with your options:

    Route::resource('photo', 'PhotoController', ['names' => [
        'create' => 'photo.build'
    ]]);

<a name="restful-naming-resource-route-parameters"></a>
### 命名資源路由參數

By default, `Route::resource` will create the route parameters for your resource routes based on the "singularized" version of the resource name. You can easily override this on a per resource basis by passing `parameters` in the options array. The `parameters` array should be an associative array of resource names and parameter names:

    Route::resource('user', 'AdminUserController', ['parameters' => [
        'user' => 'admin_user'
    ]]);

 The example above generates the following URIs for the resource's `show` route:

    /user/{admin_user}

<a name="restful-supplementing-resource-controllers"></a>
### 附加資源控制器

If you need to add additional routes to a resource controller beyond the default set of resource routes, you should define those routes before your call to `Route::resource`; otherwise, the routes defined by the `resource` method may unintentionally take precedence over your supplemental routes:

    Route::get('photos/popular', 'PhotoController@method');

    Route::resource('photos', 'PhotoController');

> {tip} Remember to keep your controllers focused. If you find yourself routinely needing methods outside of the typical set of resource actions, consider splitting your controller into two, smaller controllers.

<a name="dependency-injection-and-controllers"></a>
## 依賴注入與控制器

#### 建構子注入

The Laravel [service container](/docs/{{version}}/container) is used to resolve all Laravel controllers. As a result, you are able to type-hint any dependencies your controller may need in its constructor. The declared dependencies will automatically be resolved and injected into the controller instance:

    <?php

    namespace App\Http\Controllers;

    use App\Repositories\UserRepository;

    class UserController extends Controller
    {
        /**
         * The user repository instance.
         */
        protected $users;

        /**
         * Create a new controller instance.
         *
         * @param  UserRepository  $users
         * @return void
         */
        public function __construct(UserRepository $users)
        {
            $this->users = $users;
        }
    }

Of course, you may also type-hint any [Laravel contract](/docs/{{version}}/contracts). If the container can resolve it, you can type-hint it. Depending on your application, injecting your dependencies into your controller may provide better testability.

#### 方法注入

In addition to constructor injection, you may also type-hint dependencies on your controller's methods. A common use-case for method injection is injecting the `Illuminate\Http\Request` instance into your controller methods:

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;

    class UserController extends Controller
    {
        /**
         * Store a new user.
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            $name = $request->name;

            //
        }
    }

If your controller method is also expecting input from a route parameter, simply list your route arguments after your other dependencies. For example, if your route is defined like so:

    Route::put('user/{id}', 'UserController@update');

You may still type-hint the `Illuminate\Http\Request` and access your `id` parameter by defining your controller method as follows:

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;

    class UserController extends Controller
    {
        /**
         * Update the given user.
         *
         * @param  Request  $request
         * @param  string  $id
         * @return Response
         */
        public function update(Request $request, $id)
        {
            //
        }
    }

<a name="route-caching"></a>
## 路由快取

> {note} Closure based routes cannot be cached. To use route caching, you must convert any Closure routes to controller classes.

If your application is exclusively using controller based routes, you should take advantage of Laravel's route cache. Using the route cache will drastically decrease the amount of time it takes to register all of your application's routes. In some cases, your route registration may even be up to 100x faster. To generate a route cache, just execute the `route:cache` Artisan command:

    php artisan route:cache

After running this command, your cached routes file will be loaded on every request. Remember, if you add any new routes you will need to generate a fresh route cache. Because of this, you should only run the `route:cache` command during your project's deployment.

You may use the `route:clear` command to clear the route cache:

    php artisan route:clear
