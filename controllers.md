# 控制器

- [介紹](#introduction)
- [基礎控制器](#basic-controllers)
    - [定義控制器](#defining-controllers)
    - [控制器與命名空間](#controllers-and-namespaces)
    - [控制器的單一行為](#single-action-controllers)
- [控制器中介層](#controller-middleware)
- [資源控制器](#resource-controllers)
    - [部分資源路由](#restful-partial-resource-routes)
    - [為資源路由命名](#restful-naming-resource-routes)
    - [為資源路由命名的參數](#restful-naming-resource-route-parameters)
    - [在地化資源的 URI](#restful-localizing-resource-uris)
    - [附加資源控制器](#restful-supplementing-resource-controllers)
- [依賴注入與控制器](#dependency-injection-and-controllers)
- [路由快取](#route-caching)

<a name="introduction"></a>
## 介紹

除了在單一的 `routes.php` 檔案中定義所有的請求處理邏輯，你可能希望使用控制器類別來組織此行為。控制器可將相關的 HTTP 請求處理邏輯組成一個類別。控制器一般存放在 `app/Http/Controllers` 目錄下。

<a name="basic-controllers"></a>
## 基礎控制器

<a name="defining-controllers"></a>
### 定義控制器

這是一個基礎控制器類別的範例。所有 Laravel 控制器都應繼承基礎控制器類別，它包含在 Laravel 的預設安裝中：

    <?php

    namespace App\Http\Controllers;

    use App\User;
    use App\Http\Controllers\Controller;

    class UserController extends Controller
    {
        /**
         * Show the profile for the given user.
         *
         * @param  int  $id
         * @return Response
         */
        public function show($id)
        {
            return view('user.profile', ['user' => User::findOrFail($id)]);
        }
    }

    我們可以經由路由指定控制器行為，就像這樣：

    Route::get('user/{id}', 'UserController@show');

現在，當請求和此特定的路由 URI 相匹配時，`UserController` 類別的 `showProfile` 方法就會被執行。當然，路由的參數也會被傳遞至該方法。

> {tip} 控制器不**需要**再繼承基礎類別。然而，你將無法存取一些方便的功能，比如 `middleware`、`validate` 和 `dispatch` 這些方法。

<a name="controllers-and-namespaces"></a>
### 控制器與命名空間

有一點非常重要，那就是我們在定義控制器路由時，不需要指定完整的控制器命名空間。因為 `RouteServiceProvider` 已經把命名空間寫入路由群組中，並且會載到你的路由檔案，我們只需要指定 `App\Http\Controllers` 命名空間之後的類別名稱部分。

如果你想將控制器放到 `App\Http\Controllers` 目錄的更深處，只要指定 `App\Http\Controllers` 命名空間後面的路由。假設你的控制器是 `App\Http\Controllers\Photos\AdminController`，你應該像底下範例去註冊控制器到路由：

    Route::get('foo', 'Photos\AdminController@method');

<a name="single-action-controllers"></a>
### 控制器的單一操作

如果要定義只用來處理單一操作的控制器，可以在控制器上放置一個 `__invoke` 方法：

    <?php

    namespace App\Http\Controllers;

    use App\User;
    use App\Http\Controllers\Controller;

    class ShowProfile extends Controller
    {
        /**
         * 為給定的使用者顯示個資。
         *
         * @param  int  $id
         * @return Response
         */
        public function __invoke($id)
        {
            return view('user.profile', ['user' => User::findOrFail($id)]);
        }
    }

為只有單一操作的控制器註冊路由時，你就不用在指定方法：

    Route::get('user/{id}', 'ShowProfile');

<a name="controller-middleware"></a>
## 控制器的中介層

[中介層](/docs/{{version}}/middleware) 可以被指定在路由檔案中的控制器路由：

    Route::get('profile', 'UserController@show')->middleware('auth');

不過，在控制器建構子中指定中介層會更為方便。在控制器建構子中使用 `middleware` 方法，你可以很容易地將中介層指定給控制器行為。你甚至可以對中介層作出限制，僅將它提供給控制器類別中的某些方法。

    class UserController extends Controller
    {
        /**
         * Instantiate a new controller instance.
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

控制器還可以讓你使用閉包來註冊中介層。這提供了一個便利的方式來為單個控制器定義中介層，而不用再定義完整的中介層：

    $this->middleware(function ($request, $next) {
        // ...

        return $next($request);
    });

> {tip} 你可以將中介層指定給控制器的子操作。然而，這麽做可能會讓你的控制器變肥，所以建議你將控制器拆分成更小更多的控制器。

<a name="resource-controllers"></a>
## 資源控制器

Laravel 資源路由將典型的「CRUD」路由指定給控制器，且只使用一行的程式碼。例如，你可能希望建立一個控制器是用來處理全部的 HTTP 請求，並用來為你的應用程式儲存「照片」。使用 Artisan 指令的 `make:controller`，就可以讓我們能快速的建立像是下面的範例：

    php artisan make:controller PhotoController --resource

這個指令會在 `app/Http/Controllers/PhotoController.php` 產生一個控制器。這個控制器會包含各種用於資源操作的方法。

接下來，你可以註冊資源路由到控制器：

    Route::resource('photos', 'PhotoController');

這一條路由宣告會創建多個路由，用來處理各式各樣和資源相關的行為。同樣地，產生的控制器已有各種和這些行為繫結的方法，並包含通知你它們所處理的 URI 及 HTTP 動詞的記錄。

透過傳遞陣列給 `resources` 方法，你就可以一次註冊多個資源控制器：

    Route::resources([
        'photos' => 'PhotoController',
        'posts' => 'PostController'
    ]);

#### 由資源控制器處理的操作

Verb      | URI                  | Action       | Route Name
----------|-----------------------|--------------|---------------------
GET       | `/photos`              | index        | photos.index
GET       | `/photos/create`       | create       | photos.create
POST      | `/photos`              | store        | photos.store
GET       | `/photos/{photo}`      | show         | photos.show
GET       | `/photos/{photo}/edit` | edit         | photos.edit
PUT/PATCH | `/photos/{photo}`      | update       | photos.update
DELETE    | `/photos/{photo}`      | destroy      | photos.destroy

#### 指定資源模型

如果你正在使用路由模型綁定並想把資源控制器的方法注入模型實體，可以在你用指令產生控制器的時候，加入 `--model` 選項：

    php artisan make:controller PhotoController --resource --model=Photo

#### 偽造表單方法

由於 HTML 表單不能建立 `PUT`、`PATCH` 或 `DELETE` 請求，你會需要新增一個隱藏的 `_method` 字段來偽造一些 HTTP 動詞操作。`method_field` 輔助函式協助你建立這個字段：

    {{ method_field('PUT') }}

<a name="restful-partial-resource-routes"></a>
### 部分資源路由

在宣告資源路由的時候，你可以處理指定的控制器的部分行為，而非預設的所有行為：

    Route::resource('photo', 'PhotoController', ['only' => [
        'index', 'show'
    ]]);

    Route::resource('photo', 'PhotoController', ['except' => [
        'create', 'store', 'update', 'destroy'
    ]]);

#### API 資源路由

在宣告資源路由給 API 使用的時候，你通常會想要排除用來顯示 HTML 模板的路由，像是 `create` 和 `edit`。為了方便起見，你可以使用 `apiResource` 方法來自動排除這兩個路由：

    Route::apiResource('photo', 'PhotoController');

<a name="restful-naming-resource-routes"></a>
### 為資源路由命名

所有的資源控制器行為預設都有一路由名稱；不過你可以在選項中傳遞一個 `names` 陣列來覆寫這些名稱：

    Route::resource('photo', 'PhotoController', ['names' => [
        'create' => 'photo.build'
    ]]);

<a name="restful-naming-resource-route-parameters"></a>
### 為資源路由命名的參數

預設的 `Route::resource` 會根據「奇數」版本的資源名稱建立資源路由的路由參數。你能透過在選項陣列中傳遞 `parameters` 來輕鬆的覆蓋每個資源。`parameters` 陣列應該是與資源名稱或參數名稱有關的陣列：

    Route::resource('user', 'AdminUserController', ['parameters' => [
        'user' => 'admin_user'
    ]]);

 以上範例會為資源的 `show` 路由產生如下的 URI：

    /user/{admin_user}

<a name="restful-localizing-resource-uris"></a>
### 在地化資源的 URI

預設 `Route::resource` 會使用英文動詞來建立資源的 URI。如果你需要將 `create` 和 `edit` 行為動詞給在地化翻譯，你可以使用 `Route::resourceVerbs` 方法。這可以在你的 `AppServiceProvider` 的 `boot` 方法中完成這件事：

    use Illuminate\Support\Facades\Route;

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        Route::resourceVerbs([
            'create' => 'crear',
            'edit' => 'editar',
        ]);
    }

一旦動詞被客製化翻譯，像是 `Route::resource('fotos', 'PhotoController')`，這組資源路由就會產生以下 URI：

    /fotos/crear

    /fotos/{foto}/editar

<a name="restful-supplementing-resource-controllers"></a>
### 附加資源控制器

如果你需要向資源控制器新增額外的路由，你應該在呼叫 `Route::resource` 之前定義這些路由。否則，由 `resource` 方法定義的路由可能會意外地覆蓋你附加的路由：

    Route::get('photos/popular', 'PhotoController@method');

    Route::resource('photos', 'PhotoController');

> {tip} 請記得維持控制器的重點。如果你發現自己經常需要用到典型資源以外的方法，建議將控制器拆分為兩個較小的控制器。

<a name="dependency-injection-and-controllers"></a>
## 依賴注入與控制器

#### 建構子注入

Laravel 的[服務容器](/docs/{{version}}/container)是用來解析 Laravel 所有的控制器。因此，在建構子中，你可以對控制器可能需要的任何依賴使用型別提示。被宣告的依賴將會自動解析與注入到控制器實例中：

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

當然，你也可以對任何的 [Laravel contract](/docs/{{version}}/contracts) 使用型別提示。若容器能夠解析它，你就可以使用型別提示。根據你的應用，將你的依賴注入到控制器藉此提供更好的可測試性。

#### 方法注入

除了建構子注入外，你也可以在你的控制器方法上注入依賴。方法注入的常用案例是注入 `Illuminate\Http\Request` 實例到你的控制器方法：

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

若你的控制器方法也預期從路由參數獲得輸入值，只要在你其它的依賴之後列出路由參數即可。例如，如果你的路由被定義成這個樣子：

    Route::put('user/{id}', 'UserController@update');

你仍然可以注入 `Illuminate\Http\Request` 並透過定義控制器方法存取你的 `id` 參數，如下所示：

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

> {note} 使用閉包的路由不能被快取。要使用路由快取，你必須將任何閉包路由轉換為控制器類別。

若你的應用程式完全透過控制器使用路由，你應該考慮採用 Laravel 的路由快取。使用路由快取可以大幅降低註冊你應用程式全部的路由所需的時間。在有些情況下，你的路由註冊會被加速且快達一百倍。可以執行 Artisan 指令的 `route:cache` 來產生路由快取：

    php artisan route:cache

執行完這個指令後，你的快取路由檔案會被載到每個請求。記得如果你新增任何新的路由，你就需要產生一個新的路由快取。因此，你只應該在正式部署階段執行 `route:cache` 指令。

你可以使用 `route:clear` 指令來清除路由快取：

    php artisan route:clear
