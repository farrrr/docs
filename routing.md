# 路由

- [基本路由](#basic-routing)
- [路由參數](#route-parameters)
    - [必要參數](#required-parameters)
    - [選擇性參數](#parameters-optional-parameters)
    - [正規表達式規範](#parameters-regular-expression-constraints)
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

在大部分的應用中，你將會從定義 `routes/web.php` 檔案開始。

#### 可用的路由器方法

路由器能讓你註冊回應任何 HTTP 動詞的路由：

    Route::get($uri, $callback);
    Route::post($uri, $callback);
    Route::put($uri, $callback);
    Route::patch($uri, $callback);
    Route::delete($uri, $callback);
    Route::options($uri, $callback);

有時候你可能需要讓註冊一個路由可以對應到多個 HTTP 動詞。你可以使用 `match` 方法做到。或者甚至可以透過 `any` 方法來使用註冊路由並回應所有的 HTTP 動詞：

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

當然，你往往會需要從 URI 路由中存取參數。例如，你可能需要從 URL 取得使用者的 ID。那麼你可以透過定義路由參數來取得：

    Route::get('user/{id}', function ($id) {
        return 'User '.$id;
    });

你可以依照路由需要來定義多個路由參數：

    Route::get('posts/{post}/comments/{comment}', function ($postId, $commentId) {
        //
    });

路由參數會放在 `{}` (大括號)內且由英文字母所組成。路由參數不得包含 `-` 字元。請使用底線 (`_`) 來取代。

<a name="parameters-optional-parameters"></a>
### 選擇性的路由參數

某些時候你可能需要指定路由參數，但其存在是選擇性的。那麼你可以在參數名稱後面加上 `?` 標記。記得給該路由的對應參數一個預設值：

    Route::get('user/{name?}', function ($name = null) {
        return $name;
    });

    Route::get('user/{name?}', function ($name = 'John') {
        return $name;
    });

<a name="parameters-regular-expression-constraints"></a>
### 正規表達式規範

你可以在一個路由實例中規範使用了 `where` 方法的路由參數格式。`where` 方法可接受參數名稱，以及一個正規表達式用來規範參數：

    Route::get('user/{name}', function ($name) {
        //
    })->where('name', '[A-Za-z]+');

    Route::get('user/{id}', function ($id) {
        //
    })->where('id', '[0-9]+');

    Route::get('user/{id}/{name}', function ($id, $name) {
        //
    })->where(['id' => '[0-9]+', 'name' => '[a-z]+']);

<a name="parameters-global-constraints"></a>
#### 全域規範

如果你想讓路由參數總是被特定的正規表達式所規範，那麼你可以使用 `pattern` 方法。你可以在 `RouteServiceProvider` 中的 `boot` 方法定義該模式：

    /**
     * 定義你的路由模型綁定、模式過濾器..等等。
     *
     * @return void
     */
    public function boot()
    {
        Route::pattern('id', '[0-9]+');

        parent::boot();
    }

當該模式定義完成後，會自動地套用到所有使用到該參數名稱的路由：

    Route::get('user/{id}', function ($id) {
        // Only executed if {id} is numeric...
    });

<a name="named-routes"></a>
## 命名路由

命名路由讓你更方便地為特定路由產生 URLs 或進行重導。你可以透過在路由定義後方串連 `name` 方法來指定路由名稱：

    Route::get('user/profile', function () {
        //
    })->name('profile');

你也可以為控制器操作定義路由名稱：

    Route::get('user/profile', 'UserController@showProfile')->name('profile');

#### 對命名路由產生 URLs

一但你在特定路由中指派了路由名稱，那麼你就可以使用全域函式 `route` 來產生 URLs 或是重導：

    // 產生 URLs...
    $url = route('profile');

    // 產生重導...
    return redirect()->route('profile');

若是命名路由有定義參數，你可以在 `route` 函式傳入第二個參數。給定的參數將自動加入到 URL 中正確的位置：

    Route::get('user/{id}/profile', function ($id) {
        //
    })->name('profile');

    $url = route('profile', ['id' => 1]);

<a name="route-groups"></a>
## 路由群組

路由群組允許你共用路由屬性，例如：中介層、命名空間，你可以利用路由群組套用這些屬性到多個路由，而不需在每個路由都設定一次。共用屬性被指定為陣列格式，當作 `Route::group` 方法的第一個參數：

<a name="route-group-middleware"></a>
### 中介層

要指定中介層到所有群組內的路由，你可以在群組屬性陣列裡使用 `middleware` 參數。中介層將會依照你在列表內指定的順序執行：

    Route::group(['middleware' => 'auth'], function () {
        Route::get('/', function ()    {
            // 使用 Auth 中介層
        });

        Route::get('user/profile', function () {
            // 使用 Auth 中介層
        });
    });

<a name="route-group-namespaces"></a>
### 命名空間

另一個常見的範例是，指派相同的 PHP 命名空間中給控制器群組。你可以使用 `namespace` 數指定群組內所有控制器的命名空間：

    Route::group(['namespace' => 'Admin'], function() {
        // 控制器在 "App\Http\Controllers\Admin" 命名空間
    });

記得，預設 `RouteServiceProvider` 會在命名空間群組內導入你的路由檔案，讓你不用指定完整的 `App\Http\Controllers` 命名空間前綴就能註冊控制器路由。所以，我們只需要指定在基底 `App\Http\Controllers` 根命名空間之後的命名空間部分。

<a name="route-group-sub-domain-routing"></a>
### 子網域路由

路由群組也可以用來處理子網域。子網域可以像路由 URIs 分配路由參數，讓你在你的路由或控制器取得子網域參數。可以使用路由群組屬性陣列上的 `domain` 指定子網域變數名稱：

    Route::group(['domain' => '{account}.myapp.com'], function () {
        Route::get('user/{id}', function ($account, $id) {
            //
        });
    });

<a name="route-group-prefixes"></a>
### 路由前綴

透過路由群組陣列屬性中的 `prefix` ，在路由群組內為每個路由給定的 URI 加上前綴。例如，你可能想要在路由群組中將所有的路由 URIs 加上前綴 `admin`:

    Route::group(['prefix' => 'admin'], function () {
        Route::get('users', function ()    {
            // 符合 "/admin/users" URL
        });
    });

<a name="route-model-binding"></a>
## 路由模型綁定

當你注入一個資料模型的 ID 到路由或是控制器動作時，通常可以查詢並使用該 ID 對應的資料模型。Laravel 路由模型綁定更提供一個方便的方式來注入類別實例至你的路由中。舉例來說，除了注入一個使用者的 ID，你還可以注入對應 ID 的完整 `User` 類別實例。

<a name="implicit-binding"></a>
### 隱式綁定

Laravel 會自動解析變數名稱相符合且已定義在路由或是控制器動作中的 Eloquent 模型。下列例子：

    Route::get('api/users/{user}', function (App\User $user) {
        return $user->email;
    });

在這個例子中，由於 URI 中的 `{user}` 參數符合路由定義的 Eloquent 變數 `$user`， Laravel 將會注入與 URI 請求模型中 ID 相符合的實例。如果對應的模型實例不存在於資料庫中，那麼將自動產生一個 404 HTTP 回應。

#### 自定鍵名

如果你希望模型綁定在透過資料模型查詢資料庫時，能夠使用 `id` 以外的欄位來索引，你不妨在 Eloquent 資料模型中覆寫 `getRouteKeyName` 方法：

    /**
     * 從資料模型中取得路由鍵值
     *
     * @return string
     */
    public function getRouteKeyName()
    {
        return 'slug';
    }

<a name="explicit-binding"></a>
### 顯式綁定

要註冊顯示綁定，可使用路由器的 `model` 為特定類別指定一個參數。你應該在 `RouteServiceProvider` 類別裡的 `boot` 方法中定義顯示綁定：

    public function boot()
    {
        parent::boot();

        Route::model('user', App\User::class);
    }

接著，定義一個帶有 `{user}` 參數的路由：

    Route::get('profile/{user}', function (App\User $user) {
        //
    });

當我們將所有的 `{user}` 參數綁定到 `App\User` 資料模型，將會注入一個 `User` 到路由中。那麼舉例來說，一個來自 `profile/1` 的請求將會注入到來自資料庫中一筆 ID 為 `1` 的 `User` 實例。

如果對應的模型實例不存在於資料庫中，那麼將自動產生一個 404 HTTP 回應。

#### 自定解析邏輯

如果你想使用自己的解析邏輯，你可以使用 `Route::bind` 方法。你傳遞給 `bind` 方法的 `閉包` you pass to the `bind` 將接收 URI 參數的值，並返回應該注入到路由中的類別實例：

    public function boot()
    {
        parent::boot();

        Route::bind('user', function ($value) {
            return App\User::where('name', $value)->first();
        });
    }

<a name="form-method-spoofing"></a>
## 表單方法欺騙

HTML 表單並沒有支援 `PUT`、`PATCH` 或 `DELETE` 動作。所以在定義 `PUT`、`PATCH` 或是 `DELETE` 路由，且在 HTML 表單中被呼叫的時候，你將需要在表單中增加隱藏的 `_method` 欄位。隨著 `_method` 欄位送出的值將被視為 HTTP 請求方法使用：

    <form action="/foo/bar" method="POST">
        <input type="hidden" name="_method" value="PUT">
        <input type="hidden" name="_token" value="{{ csrf_token() }}">
    </form>

你不妨使用 `method_field` 輔助函式來產生 `_method` 欄位：

    {{ method_field('PUT') }}

<a name="accessing-the-current-route"></a>
## 存取當前路由

你不妨使用 `current`、`currentRouteName`、以及 `currentRouteAction` 這些 `Route` facade 中的方法來取得當前路由請求的資訊：

    $route = Route::current();

    $name = Route::currentRouteName();

    $action = Route::currentRouteAction();

請參閱 API 文件來了解 [路由 facade 基礎類別](http://laravel.com/api/{{version}}/Illuminate/Routing/Router.html) 以及 [路由實例](http://laravel.com/api/{{version}}/Illuminate/Routing/Route.html) 以查看所有可用的方法。
