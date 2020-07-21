# URL 產生

- [介紹](#introduction)
- [基本用法](#the-basics)
    - [產生基本的 URL](#generating-basic-urls)
    - [存取當前的 URL](#accessing-the-current-url)
- [已命名路由的 URL](#urls-for-named-routes)
    - [具備簽章的 URL](#signed-urls)
- [控制器行為的 URL](#urls-for-controller-actions)
- [預設值](#default-values)

<a name="introduction"></a>
## 介紹

Laravel 提供幾個輔助函式來協助你產生 URL。這些有助於在模板和 API 回應中建構連結，或產生重導回應到應用程式的另一個部分的時候。

<a name="the-basics"></a>
## 基本用法

<a name="generating-basic-urls"></a>
### 產生基本的 URL

`url` 輔助函式可被用於應用程式的任何一個 URL。被產生的 URL 會自動使用當前請求所用的傳輸協定（HTTP 或 HTTPS）和主機：

    $post = App\Post::find(1);

    echo url("/posts/{$post->id}");

    // http://example.com/posts/1

<a name="accessing-the-current-url"></a>
### 存取當前的 URL

如果不提供路徑到 `url` 輔助函式，會回傳 `Illuminate\Routing\UrlGenerator` 實例，這可以讓你去存取關於目前 URL 的資訊：

    // 未使用查詢字串來取得當前 URL...
    echo url()->current();

    // 使用查詢字串來取得當前 URL...
    echo url()->full();

    // 取得上一個請求的完整 URL...
    echo url()->previous();

上面這些方法都可以透過 `URL` [facade](/docs/{{version}}/facades) 來存取：

    use Illuminate\Support\Facades\URL;

    echo URL::current();

<a name="urls-for-named-routes"></a>
## 已命名路由的 URL

`route` 輔助函式可被用於產生 URL 到被命名的路由。已命名的路由可以讓你產生 URL，還不會影響到實際在路由上定義的 URL。因此，如果該路由的 URL 有被異動，就不用在去修改你的 `route` 函式呼叫。例如，設想你的應用程式有一個像是下面範例所定義的路由：

    Route::get('/post/{post}', function () {
        //
    })->name('post.show');

要產生 URL 到這個路由，你可以使用 `route` 輔助函式，就像是：

    echo route('post.show', ['post' => 1]);

    // http://example.com/post/1

你通常會使用 [Eloquent 模型](/docs/{{version}}/eloquent)的主鍵來產生 URL。出於這個緣故，你可以將 Eloquent 模型作為參數值來傳入。`route` 輔助函式會自動的取出模型的主鍵：

    echo route('post.show', ['post' => $post]);

The `route` helper may also be used to generate URLs for routes with multiple parameters:

    Route::get('/post/{post}/comment/{comment}', function () {
        //
    })->name('comment.show');

    echo route('comment.show', ['post' => 1, 'comment' => 3]);

    // http://example.com/post/1/comment/3

<a name="signed-urls"></a>
### 具備簽章的 URL

Laravel 能輕易的將已命名的路由建立「具備簽章」的 URL。這類 URL 的查詢字串會補上「signature」的雜湊值，使 Laravel 能夠加以驗證這些 URL 從建立以來未被竄改過。具備簽章的 URL 適用於需要禁止 URL 被竄改且可被公開存取的路由。

像是，你可以使用具備簽章的 URL 來實作電子信件中的「取消訂閱」的公開連結。使用 `URL` facade 的 `signedRoute` 方法即可建立具備簽章的 URL：

    use Illuminate\Support\Facades\URL;

    return URL::signedRoute('unsubscribe', ['user' => 1]);

若你想要產生一組有效期限簽章的路由 URL，你可以使用 `temporarySignedRoute` 方法：

    use Illuminate\Support\Facades\URL;

    return URL::temporarySignedRoute(
        'unsubscribe', now()->addMinutes(30), ['user' => 1]
    );

#### 驗證具備簽章的路由請求

要驗證傳入的請求的簽章是否有效，你應該在傳入的 `Request` 上呼叫 `hasValidSignature` 方法：

    use Illuminate\Http\Request;

    Route::get('/unsubscribe/{user}', function (Request $request) {
        if (! $request->hasValidSignature()) {
            abort(401);
        }

        // ...
    })->name('unsubscribe');

或者，你可以指派 `Illuminate\Routing\Middleware\ValidateSignature` 中介層到該路由。如果它還不存在，你應該在你的 HTTP kernel 的 `routeMiddleware` 陣列中加入這個中介層的鍵：

    /**
     * 應用程式的路由中介層
     *
     * 這些中介層可以設定在群組上或個別使用
     *
     * @var array
     */
    protected $routeMiddleware = [
        'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,
    ];

一但你在 kernel 內註冊了該中介層，你就可以附加到路由上。如果傳入的請求沒有有效的簽章，該中介層將會自動回傳 `403` 的錯誤回應：

    Route::post('/unsubscribe/{user}', function (Request $request) {
        // ...
    })->name('unsubscribe')->middleware('signed');
        
<a name="urls-for-controller-actions"></a>
## 控制器行為的 URL

`action` 函式為給定控制器行為來產生一組 URL。你不需要再傳入控制器的完整命名空間。反而是傳入相對於 `App\Http\Controllers` 命名空間的控制器類別名稱的相對路徑：

    $url = action('HomeController@index');

你也可以使用「回呼」陣列語法來引用行為：

    use App\Http\Controllers\HomeController;

    $url = action([HomeController::class, 'index']);

如果控制器方法接受路由參數，你可以將他們作為第二個參數來傳入該函式：

    $url = action('UserController@profile', ['id' => 1]);

<a name="default-values"></a>
## 預設值

因為某些應用程式的關係，你可能希望為某些 URL 參數指定請求範圍的預設值。例如，設想你的許多路由定義了 `{locale}` 參數：

    Route::get('/{locale}/posts', function () {
        //
    })->name('post.index');

如果每次呼叫 `route` 輔助函式都要傳入 `locale`，會使開發變得很繁瑣。所以，你可以使用 `URL::defaults` 方法來定義這個參數的預設值，使該參數總是在當前請求期間被應用。你可能希望從[路由中介層](/docs/{{version}}/middleware#assigning-middleware-to-routes)中呼叫這個方法，這樣就可以存取當前的請求：

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Support\Facades\URL;

    class SetDefaultLocaleForUrls
    {
        public function handle($request, Closure $next)
        {
            URL::defaults(['locale' => $request->user()->locale]);

            return $next($request);
        }
    }

一旦 `locale` 參數的預設值被設定，你就不需要在透過 `route` 輔助函式來產生 URL 時傳入它的值了。