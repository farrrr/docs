# Authentication

- [介紹](#introduction)
    - [資料庫注意事項](#introduction-database-considerations)
- [認證快速入門](#authentication-quickstart)
    - [路由](#included-routing)
    - [視圖](#included-views)
    - [認證](#included-authenticating)
    - [取得已認證的使用者](#retrieving-the-authenticated-user)
    - [保護路由](#protecting-routes)
    - [認證限制](#login-throttling)
- [手動認證使用者](#authenticating-users)
    - [記住使用者](#remembering-users)
    - [其他認證方法](#other-authentication-methods)
- [HTTP 基礎認證](#http-basic-authentication)
    - [無狀態 HTTP 基礎認證](#stateless-http-basic-authentication)
- [社群認證](https://github.com/laravel/socialite)
- [新增客製化守衛](#adding-custom-guards)
- [新增客製化使用者守衛](#adding-custom-user-providers)
    - [提供使用者的 Contract](#the-user-provider-contract)
    - [認證用的 Contract](#the-authenticatable-contract)
- [事件](#events)

<a name="introduction"></a>
## 介紹

> {溫馨提醒} **想快速入門？** 只要在新建的 Laravel 執行 `php artisan make:auth` 和 `php artisan migrate` 指令。 然後，瀏覽 `http://your-app.dev/register` 這個網址或重新定義你想要的路由位址。這兩個指令就可以快速地建立整個認證系統！

Laravel 實作認證功能非常簡單。事實上，幾乎所有東西都已經幫你設定好了。設定認證系統的檔案在 `config/auth.php` ，其中還包含了幾個好用的選項用來調整認證系統。

Laravel 的認證系統核心主要由「守衛」與「提供者」組成。守衛定義了在每個請求中，如何與使用者進行身份認證。舉例來說， Laravel 內建的 `session` 守衛會使用 session 與 cookies 來維護認證狀態。

「提供者」定義如何從資料庫中取得使用者資料。 Laravel 內建支援使用 Eloquent 和資料庫查詢產生器來取得使用者資料。然而，你可以依據應用程式的需求來定義額外的「提供者」。

如果你聽到這裡就覺得很混亂的話，別擔心！大多數的應用程式都不太需要修改預設的設定。

<a name="introduction-database-considerations"></a>
### 資料庫注意事項

預設的 Laravel 會在 `app` 資料夾中內建一個 `App\User` [Eloquent 模型](/docs/{{version}}/eloquent) 。這個模型預設只用 Eloquent 認證驅。如果你的應用程式沒有使用 Eloquent，你可以改用 `database` 這個使用了 Laravel 查詢生成器的認證驅動。

為 `App\User` 模型建立資料庫結構時，請確保密碼欄位長度至少有60字元長。建議字元長度為255字元長。

還有，你應該確認你的 `users` 資料表(或同等意義的)要有一個 nullable、100字元長的欄位給 `remember_token`，這個欄位將會用來儲存「記住我」的 session 標記。

<a name="authentication-quickstart"></a>
## 認證快速入門

Laravel 內建幾個認證控制器，它們被放置在 `App\Http\Controllers\Auth` 這個命名空間。 `RegisterController` 處理新使用者的註冊； `LoginController` 處理使用者認證； `ForgotPasswordController` 處理用於重置密碼的e-mail連結，還有 `ResetPasswordController` 負責重置密碼的程式邏輯。這些控制器都使用了 trait 來包含所需要的方法，對於大多數的應用程式而言，你不太需要修改這些控制器。

<a name="included-routing"></a>
### 路由

Laravel 提供一個簡單的指令來快速建立所有認證所需的路由和視圖：

    php artisan make:auth

這個指令應該只被用於全新的應用程式，且還會安裝註冊和登入頁面與所有認證相關的路由設定。同時還會產生 `HomeController` 來處理儀表板的登入請求。

<a name="included-views"></a>
### 視圖

正如上述， `php artisan make:auth` 這個指令會建立所有認證相關的視圖，並放在 `resources/views/auth` 這個資料夾。

 `make:auth` 這個指令還會建立 `resources/views/layouts` 資料夾，內含一組預設的版面設計給你的應用程式使用。這些預設的視圖都採用 Bootstrap CSS 框架，但是你可以根據實際需求來重新客製化。

<a name="included-authenticating"></a>
### 認證

現在你已經為認證系統設定好了路由與頁面視圖，你可以開始讓你的新使用者體驗註冊與認證登入你的應用程式啦！你可以簡單的在瀏覽器存取你的應用程式，在認證控制器中，已有程式邏輯（通過各自的 traits）去認證現有的使用者並將新使用者資料儲存在資料庫中。

#### 自訂路徑

當使用者成功通過認證，他們將會重導至 `/home` 這個URI。你能透過修改 `LoginController`、 `RegisterController`和 `ResetPasswordController` 這些控制器中的 `redirectTo` 屬性來重新選擇想要重導的位置：

    protected $redirectTo = '/';

如果重導的路徑需要運用特定的程式邏輯，你只要定義 `redirectTo` 這個方法來取代 `redirectTo` 這個屬性即可：

    protected function redirectTo()
    {
        return '/path';
    }

> {溫馨提醒} 這個 `redirectTo` 方法將優先於 `redirectTo` 屬性。

#### 自訂使用者名稱

 Laravel 預設是使用 `email` 去做認證。如果想要修改，你可以自行在 `LoginController` 這個控制器中定義 `username` 屬性內容：

    public function username()
    {
        return 'username';
    }

#### 自訂「守衛」

你可以自行定義認證與註冊用的「守衛」。要實現這一功能，需要再 `LoginController`、`RegisterController`和 `ResetPasswordController` 這些控制器中定義 `guard` 方法。這個方法會回傳一個「守衛」實例：

    use Illuminate\Support\Facades\Auth;

    protected function guard()
    {
        return Auth::guard('guard-name');
    }

#### 自訂驗證與儲存

要修改一個新使用者註冊時所需填寫的表單欄位，或是自訂如何將使用者的紀錄新增到資料庫的方式，你可以修改 `RegisterController` 這個類別。這個類別負責驗證與建立新的使用者。

`RegisterController` 的 `validator` 方法包含了對於新使用者的驗證規格，你可以任意修改這個方法。

`RegisterController` 的 `create` 方法負責使用 [Eloquent ORM](/docs/{{version}}/eloquent) 建立新的 `App\User` 記錄到你的資料庫。你可以根據實際資料庫需求任意修改這個方法。

<a name="retrieving-the-authenticated-user"></a>
### 取得已認證的使用者

你可以使用 `Auth` facade 來存取認證使用者:

    use Illuminate\Support\Facades\Auth;

    // 取得目前已驗證的使用者...
    $user = Auth::user();

    // 取得目前已認證的使用者 ID ...
    $id = Auth::id();

另外，你也可以從 `Illuminate\Http\Request` 實例存取已認證的使用者。要記得，型別提示的類別將會自動注入到你的控制的方法中：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;

    class ProfileController extends Controller
    {
        /**
         * 更新使用者資料
         *
         * @param  Request  $request
         * @return Response
         */
        public function update(Request $request)
        {
            // $request->user() 回傳已認證的使用者實例...
        }
    }

#### 確認目前的使用者是否通過認證

為了確定使用者是否已經登入，你可以使用 `Auth` facade 的 `check` 方法，如果使用者已被認證過，將回傳 `true`：

    use Illuminate\Support\Facades\Auth;

    if (Auth::check()) {
        // 如果使用者已登入就...
    }

> {溫馨提醒} 即使可以使用 `check` 方法來檢查使用者是否有通過認證，但還是建議你在使用者存取控制器或路由之前，使用「中介層」來過濾使用者的認證狀態。想知道更多資訊，可以看看[保護路由](/docs/{{version}}/authentication#protecting-routes)這個文件（其實就在下面）。

<a name="protecting-routes"></a>
### 保護路由

[路由中介層](/docs/{{version}}/middleware)能使用於允許限定已認證的使用者存取指定的路由。 Laravel 內建的 `auth` 中介層被定義在 `Illuminate\Auth\Middleware\Authenticate` 這個路徑。由於這個中介層已註冊在 HTTP 的核心中，所以你只要將中介層附加到路由定義中：

    Route::get('profile', function () {
        // 只有通過認證的使用者可以...
    })->middleware('auth');

當然，如果正在使用[控制器](/docs/{{version}}/controllers)，你可以在建構子呼叫 `middleware` 這個方法，而不用在路由中定義它：

    public function __construct()
    {
        $this->middleware('auth');
    }

#### 指定「守衛」

當你將 `auth` 這個中介層加入到某個路由，你也可以指定要哪個「守衛」來認證使用者。指定的「守衛」必須對應到 `auth.php` 這個設定檔中 `guards` 陣列的其中一個鍵名：

    public function __construct()
    {
        $this->middleware('auth:api');
    }

<a name="login-throttling"></a>
### 認證限制

如果你使用 Laravel 內建的 `LoginController` 類別， `Illuminate\Foundation\Auth\ThrottlesLogins` trait 已放入你的控制器中。在預設的情況下，使用者如果嘗試登入多次都無法成功，那麼該使用者在一分鐘內無法再進行登入，且這個限制是根據使用者的名稱、 email 和 IP 位置。

<a name="authenticating-users"></a>
## 手動認證使用者

當然，如果你不想使用 Laravel 內建的認證控制器，且想刪除這些控制器，你只需要直接使用 Laravel 的認證類別來處理使用者認證。別擔心！這很簡單唷！

我們將使用 [facade](/docs/{{version}}/facades)的 `Auth` 存取 Laravel 認證服務，所以我們需要確認使否導入 facade 的 `Auth` 到類別。接下來，讓我們看看 `attempt` 這個方法：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\Auth;

    class LoginController extends Controller
    {
        /**
         * 嘗試處理認證
         *
         * @return Response
         */
        public function authenticate()
        {
            if (Auth::attempt(['email' => $email, 'password' => $password])) {
                // 如果認證通過就...
                return redirect()->intended('dashboard');
            }
        }
    }

`attempt` 這個方法會接受一個鍵值對陣列作為第一個參數。這個陣列的值會用來尋找資料庫裡的使用者資料，所以在上面的範例中，會藉由取得使用者的 e-mail欄位去尋找使用者的資料，如果找到了相對應的 e-mail ，就會用資料庫已雜湊化的使用者密碼與陣列中的也雜湊化的密碼做比對，一旦比對成功，就會啟動一組以認證的 session 給使用者。

如果認證成功， `attempt` 這個方法將會回傳 `true` 。反之，則回傳 `false` 。

在被認證用的中介層過濾前，`intended` 這個方法可以將使用者導回他們嘗試存取的頁面。如果導回的頁面不存在，則可以傳導到指定的頁面。

#### 指定額外的條件

如果你希望，你也可以加入除了 Email 和 密碼以外的條件到認證查詢。舉例來說，我們要確認使用者是否標記「 active 」:

    if (Auth::attempt(['email' => $email, 'password' => $password, 'active' => 1])) {
        // 如果這個使用者的活動是存在且有效的話...
    }

> {注意} 在這些例子中， `email` 不是一個必要的選項，它只被用來當作範例。你應該使用你的資料庫中與「 username 」相同意義的欄位名稱。


#### 存取指定「守衛」實例

你可以在使用 facade 的 `Auth` 時使用 `guard` 來指定「守衛」實例。這允許你在管理應用程式的不同部分時，可以使用不同的認證模組或使用者的資料表。

`guard` 這個方法所指定的守衛名稱必須對應到 `auth.php` 這個設定檔中 guards 陣列的其中一個鍵名：

    if (Auth::guard('admin')->attempt($credentials)) {
        //
    }

#### 登出

為了讓使用者登出，你可以使用 facade `Auth` 的 `logout` 這個方法。這個方法會清除使用者在 session 中所有認證相關關的資料：

    Auth::logout();

<a name="remembering-users"></a>

### 記住使用者

如果你想要提供「記住我」的功能，你可以傳入一個布林值到 `attempt` 這個方法的第二個參數，這會維持使用者的 session 存在直到使用者手動登出。你的 `users` 資料表必須要包含一個 `remember_token` 欄位，這將用來儲存「記住我」的標記。

    if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
        // 如果這個使用者已被記住就...
    }

> {溫馨提醒} 如果你使用 Laravel 內建的 `LoginController` ，則「記住」使用者的程式邏輯已經由控制器使用的 traits 實作。

如果你「已經記住」使用者，你可以使用 `viaRemember` 這個方法來確認這個使用者是否使用「記住我」 cookie 來做認證：

    if (Auth::viaRemember()) {
        //
    }

<a name="other-authentication-methods"></a>
### 其他認證方法

#### 用使用者實例做認證Authenticate A User Instance

如果你需要使用存在的使用者實例來登入，你需要呼叫 `login` 這個方法，並傳入使用實例，這個物件必須實作 `Illuminate\Contracts\Auth\Authenticatable` （[contract](/docs/{{version}}/contracts)）。當然，內建的 `App\User` 模型已經實作了這個介面：

    Auth::login($user);

    // 登入並「記住」給使用者...
    Auth::login($user, true);

當然，你可以指定「守衛」實例：

    Auth::guard('admin')->login($user);

#### 用使用者 ID 做認證

如果你需要用使用者 ID 來登入，你需要使用 `loginUsingId` 方法，這個方法只接受要登入的使用者的主鍵：

    Auth::loginUsingId(1);

    // 登入並「記住」給使用者...
    Auth::loginUsingId(1, true);

#### 一次性的使用者認證

你可以使用 `once` 方法來只針對一次的請求來認證使用者，沒有任何的 session 或 cookie 會被使用，這個對於建議無狀態的 API 非常的好用：

    if (Auth::once($credentials)) {
        //
    }

<a name="http-basic-authentication"></a>
## HTTP 基礎認證

[HTTP 基礎認證](https://en.wikipedia.org/wiki/Basic_access_authentication)提供一個快速的方法來認證使用者，不需要任何的「登入」頁面。首先，先將 `auth.basic` [中介層](/docs/{{version}}/middleware)加入到你的路由上。 `auth.basic` 這個中介層已經內建在 Laravel 裡面了，所以你不需要在額外定義它：

    Route::get('profile', function () {
        // 只有被認證的使用者可以進入...
    })->middleware('auth.basic');

一但中介層被加入到路由，當使用瀏覽器進入這個路由時，會主動提示你需要提供憑證。預設情況下， `auth.basic` 這個中介層會使用 `email` 欄位當作「使用者名稱」。

#### FastCGI的注意事項

如果你使用了 PHP FastCGI ， HTTP 基礎認證可能會無法正常運作，你需要將下面的幾行設定加到你的 `.htaccess` 檔案中：

    RewriteCond %{HTTP:Authorization} ^(.+)$
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

<a name="stateless-http-basic-authentication"></a>
### 無狀態 HTTP 基礎認證

你也可以使用 HTTP 基礎認證而不用在 session 中設定使用者認證用的 cookie ，這個功能對 API 認證來說非常有用。為了達到目的，[定義一個中介層](/docs/{{version}}/middleware)會呼叫 `onceBasic` 方法。如果沒有任何回應從 `onceBasic` 方法返回的話，這個請求就會進一步傳進應用程式中：

    <?php

    namespace Illuminate\Auth\Middleware;

    use Illuminate\Support\Facades\Auth;

    class AuthenticateOnceWithBasicAuth
    {
        /**
         * Handle an incoming request.
         *
         * @param  \Illuminate\Http\Request  $request
         * @param  \Closure  $next
         * @return mixed
         */
        public function handle($request, $next)
        {
            return Auth::onceBasic() ?: $next($request);
        }

    }

接著, [註冊這個路由](/docs/{{version}}/middleware#registering-middleware)，並將它增加到路由上：

    Route::get('api/user', function () {
        // 只有被認證的使用者可以進入...
    })->middleware('auth.basic.once');

<a name="adding-custom-guards"></a>
## 新增客製化守衛

你可以使用 facade `Auth` 的 `extend` 方法來定義屬於你自己的身份認證守衛。你需要在 [服務提供者](/docs/{{version}}/providers)中呼叫 `provider`。 由於 Laravel 已內建了 `AuthServiceProvider`，所以我們可以把程式碼放到這個檔案裡：

    <?php

    namespace App\Providers;

    use App\Services\Auth\JwtGuard;
    use Illuminate\Support\Facades\Auth;
    use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

    class AuthServiceProvider extends ServiceProvider
    {
        /**
         * 註冊任何應用程式的認證與授權服務
         *
         * @return void
         */
        public function boot()
        {
            $this->registerPolicies();

            Auth::extend('jwt', function ($app, $name, array $config) {
                // 回傳 Illuminate\Contracts\Auth\Guard 的實例...

                return new JwtGuard(Auth::createUserProvider($config['provider']));
            });
        }
    }

你可以看到上面的範例中，傳遞給 `extend` 方法的回呼函示必須要回傳 `Illuminate\Contracts\Auth\Guard` 的實例。你將需要時做這個介面來定義一個客製化的守衛。一但你定義完後，你可以在 `auth.php` 這個設定檔設定 `guards` 並開始使用：

    'guards' => [
        'api' => [
            'driver' => 'jwt',
            'provider' => 'users',
        ],
    ],

<a name="adding-custom-user-providers"></a>
## 新增客製化使用者守衛

如果你不使用傳統關聯式資料庫去存放你的使用者資料，你將需要擴充 Laravel 來新增你自己的認證使用者的提供者。我們將使用 facade `Auth` 的 `provider` 方法來定義客製化提供者：

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Auth;
    use App\Extensions\RiakUserProvider;
    use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

    class AuthServiceProvider extends ServiceProvider
    {
        /**
         * 註冊任何應用程式的認證與授權服務
         *
         * @return void
         */
        public function boot()
        {
            $this->registerPolicies();

            Auth::provider('riak', function ($app, array $config) {
                // 回傳 Illuminate\Contracts\Auth\UserProvider 的實例...

                return new RiakUserProvider($app->make('riak.connection'));
            });
        }
    }

在你使用 `provider` 方法註冊了提供者後，你就可以在 `auth.php` 這個設定檔中改用新的「使用者提供者」。首先，定義一個使用你的新驅動 `provider`：

    'providers' => [
        'users' => [
            'driver' => 'riak',
        ],
    ],

最後，你可以在 `guards` 設定中使用這個服務提供者：

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
    ],

<a name="the-user-provider-contract"></a>
### 提供使用者的 Contract

`Illuminate\Contracts\Auth\UserProvider` 的實例只負責取得 `Illuminate\Contracts\Auth\Authenticatable` 的實例且不受限於永久儲存系統，例如 MySQL 、 Riak 等等。這兩個介面都允許 Laravel 認證機制不用管使用者資料是如何被儲存或用什麼類型來代表它，它都會持續運作。

讓我們看看這個 `Illuminate\Contracts\Auth\UserProvider` contract:

    <?php

    namespace Illuminate\Contracts\Auth;

    interface UserProvider {

        public function retrieveById($identifier);
        public function retrieveByToken($identifier, $token);
        public function updateRememberToken(Authenticatable $user, $token);
        public function retrieveByCredentials(array $credentials);
        public function validateCredentials(Authenticatable $user, array $credentials);

    }

`retrieveById` 函式通常接收一個代表使用者的值，例如 MySQL 中自動增加的 ID 。這個方法應該要取得並回傳比對這個 ID 的 `Authenticatable` 的實例。

`retrieveByToken` 函式藉由使用者獨特的 `$identifier`和儲存在 `remember_token` 欄位的「記住我」 `$token` 來取得使用者。如同之前的方法， `Authenticatable` 的實力應該被回傳。

`updateRememberToken` 方法使用新的 `$token` 更新了 `$user` 的 `remember_token` 欄位。這個新的標記（當使用「記住我」嘗試登入成功時)，或是 null （當使用者登出時）。

在嘗試登入應用程式時，`retrieveByCredentials` 方法會接收 `Auth::attempt` 方法給的憑證陣列。這個方法應當要「查詢」底層永久儲存的資料來比對使用者的憑證。通常，這個方法會利用 `$credentials['username']` 來執行一個帶著「where」條件地查詢。接著這個方法需要會傳 `Authenticatable` 實例。 **這個方法不應該企圖做任何密碼的驗證或認證！**

`validateCredentials` 方法應該要拿 `$user` 與 `$credentials` 進行核對來認證這個使用者。例如，這個方法應當使用 `Hash::check` 來核對 `$user->getAuthPassword()` 與 `$credentials['password']` 的值。這個方法應當會傳 `true` 或 `false` 來確認密碼使否正確。

<a name="the-authenticatable-contract"></a>
### 認證用的 Contract

現在我們已經介紹了 `UserProvider` 的每個方法，讓我們看一下 `Authenticatable` contract 。請記得， `UserProvider` 的 `retrieveById` 和 `retrieveByCredentials` 方法需要回傳 `Authenticatable` 的實例：

    <?php

    namespace Illuminate\Contracts\Auth;

    interface Authenticatable {

        public function getAuthIdentifierName();
        public function getAuthIdentifier();
        public function getAuthPassword();
        public function getRememberToken();
        public function setRememberToken($value);
        public function getRememberTokenName();

    }

這個介面很簡單！ `getAuthIdentifierName` 方法會回傳「使用者名稱的主鍵」，和利用 `getAuthIdentifier` 方法回傳「使用者的主鍵」。在 MySQL，這個主鍵是指自動增加的主鍵。而`getAuthPassword` 應該要回傳使用者雜湊後的密碼。這個介面允許認證系統與任何使用者類別合作，不用管你在使用何種 ORM 或是儲存抽象層。 Laravel 的 `app` 資料夾中內建了實作這個介面的 `User` 類別，所以你可以觀察這個類別作為實作的範例。

<a name="events"></a>
## 事件

Laravel 在認證過程中觸發了許多[事件](/docs/{{version}}/events)，你可以在 `EventServiceProvider` 添加監聽器來監聽事件：

    /**
     * 添加監聽器來監聽應用程式
     *
     * @var array
     */
    protected $listen = [
        'Illuminate\Auth\Events\Registered' => [
            'App\Listeners\LogRegisteredUser',
        ],

        'Illuminate\Auth\Events\Attempting' => [
            'App\Listeners\LogAuthenticationAttempt',
        ],

        'Illuminate\Auth\Events\Authenticated' => [
            'App\Listeners\LogAuthenticated',
        ],

        'Illuminate\Auth\Events\Login' => [
            'App\Listeners\LogSuccessfulLogin',
        ],

        'Illuminate\Auth\Events\Failed' => [
            'App\Listeners\LogFailedLogin',
        ],

        'Illuminate\Auth\Events\Logout' => [
            'App\Listeners\LogSuccessfulLogout',
        ],

        'Illuminate\Auth\Events\Lockout' => [
            'App\Listeners\LogLockout',
        ],
    ];
