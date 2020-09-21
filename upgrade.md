# 升級指南

- [從 5.8 升級到 6.0](#upgrade-6.0)

<a name="high-impact-changes"></a>
## 高度影響的變更

<div class="content-list" markdown="1">
- [授權資源與 `viewAny`](#authorized-resources)
- [字串與陣列的輔助函式](#helpers)
</div>

<a name="medium-impact-changes"></a>
## 中度影響的變更

<div class="content-list" markdown="1">
- [Carbon 1.x 將停止維護](#carbon-support)
- [Redis 預設客戶端](#redis-default-client)
- [資料庫的 `Capsule::table` 方法](#capsule-table)
- [Eloquent 可陣列化與 `toArray`](#eloquent-to-array)
- [Eloquent `BelongsTo::update` 方法](#belongs-to-update)
- [Eloquent 主鍵類別](#eloquent-primary-key-type)
- [本地化的 `Lang::trans` 與 `Lang::transChoice` 方法](#trans-and-trans-choice)
- [本地化的 `Lang::getFromJson` 方法](#get-from-json)
- [隊列重試次數](#queue-retry-limit)
- [重寄 email 驗證路由](#email-verification-route)
- [Email 認證路由變更](#email-verification-route-change)
- [`Input` Facade](#the-input-facade)
</div>

<a name="upgrade-6.0"></a>
## 從 5.8 升級到 6.0

#### 預估升級時間：一小時

> {note} 我們嘗試記錄每個重大變更。由於有些重大的變更是在框架最隱密的地方，但實際上只有一小部分的變更會影響你的應用程式。

### PHP 7.2 Required

**影響程度：中**

PHP 7.1 將於 2019 年 12 月起停止維護。因此，Laravel 6.0 需要使用 PHP 7.2 或更高的版本。

<a name="updating-dependencies"></a>
### 升級依賴項目

在 `composer.json` 檔升級你的 `laravel/framework` 依賴項目到 `^6.0`。

接著，請檢查你的應用程式所使用的第三方套件的版本是否能支援 Laravel 6。

### 授權

<a name="authorized-resources"></a>
#### 授權資源與 `viewAny`

**影響程度：高**

使用 `authorizeResource` 方法來附加授權功能的控制器，現在需要定義 `viewAny` 方法來讓使用者存取控制器的 `index` 方法。不然，在呼叫控制器的 `index` 方法時會因未經授權而被拒絕。

#### 授權回應

**影響程度：低**

`Illuminate\Auth\Access\Response` 類別的建構子參數已有變更。請更新相對應的程式碼。如果你沒有手動建構授權回應且在原則中使用 `allow` 與 `deny` 實例方法，則不需要做修改：

    /**
     * 建立新回應。
     *
     * @param  bool  $allowed
     * @param  string  $message
     * @param  mixed  $code
     * @return void
     */
    public function __construct($allowed, $message = '', $code = null)

#### 回傳「拒絕」的回應

**影響程度：低**

在 Laravel 之前的版本，你不需要從原則方法中回傳 `deny` 方法的結果，因為這會拋出異常。然而，根據 Laravel 最新文件，你必須從原則中回傳 `deny` 方法的結果：

    public function update(User $user, Post $post)
    {
        if (! $user->role->isEditor()) {
            return $this->deny("你必須是編輯者才能編輯此文章。")
        }

        return $user->id === $post->user_id;
    }

<a name="auth-access-gate-contract"></a>
#### `Illuminate\Contracts\Auth\Access\Gate` Contract

**影響程度：低**

`Illuminate\Contracts\Auth\Access\Gate` Contract 可以接收新的 `inspect` 方法。如果你正在手動實作這個介面，請加入這個方法到實作中。

### Carbon

<a name="carbon-support"></a>
#### Carbon 1.x 將停止維護

**影響程度：中**

Carbon 1.x [已停止維護](https://github.com/laravel/framework/pull/28683)，因為已經接近它的維護週期的期限。請升級到 Carbon 2.0。

### 設定

#### `AWS_REGION` 環境變數

**影響程度：非必要**

如果你計畫使用 [Laravel Vapor](https://vapor.laravel.com)，請將 `config` 目錄所有出現過 `AWS_REGION` 改為 `AWS_DEFAULT_REGION`。還有，請更新 `.env` 檔的環境變數名稱。

<a name="redis-default-client"></a>
#### Redis 預設客戶端

**影響程度：中**

預設的 Redis 客戶端已從 `predis` 改成 `phpredis`。
The default Redis client has changed from `predis` to `phpredis`. In order to keep using `predis`, ensure the `redis.client` configuration option is set to `predis` in your `config/database.php` configuration file.

<a name="dynamodb-cache-store"></a>
#### DynamoDB 快取儲存方式

**影響程度：非必要**

如果你計畫使用 [Laravel Vapor](https://vapor.laravel.com)，請加入 `dynamodb` 儲存方式到 `config/cache.php` 檔案。 

    <?php
    return [
        ...
        'stores' => [
            ...
            'dynamodb' => [
                'driver' => 'dynamodb',
                'key' => env('AWS_ACCESS_KEY_ID'),
                'secret' => env('AWS_SECRET_ACCESS_KEY'),
                'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
                'table' => env('DYNAMODB_CACHE_TABLE', 'cache'),
                'endpoint' => env('DYNAMODB_ENDPOINT'),
            ],
        ],
        ...
    ];

<a name="sqs-environment-variables"></a>
#### SQS 環境變數

**影響程度：非必要**

如果你計畫使用 [Laravel Vapor](https://vapor.laravel.com)，請更新 `config/queue.php` 檔案的 `sqs` 連線方式的環境變數。 

    <?php
    return [
        ...
        'connections' => [
            ...
            'sqs' => [
                'driver' => 'sqs',
                'key' => env('AWS_ACCESS_KEY_ID'),
                'secret' => env('AWS_SECRET_ACCESS_KEY'),
                'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
                'queue' => env('SQS_QUEUE', 'your-queue-name'),
                'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
            ],
        ],
        ...
    ];

### Database

<a name="capsule-table"></a>
#### Capsule `table` 方法

**影響程度：中**

> {note} 這個改變只應用在使用 `illuminate/database` 依賴項目但沒使用 Laravel 的應用程式。

`Illuminate\Database\Capsule\Manager` 類別的 `table` 方法的參數已更新了第二個參數作為資料表別名。如果你正在 Laravel 之外的應用程式使用 `illuminate/database`，請更新任何呼叫過這個方法所對應的參數：

    /**
     * 取得優雅的查詢建構器的實例。
     *
     * @param  \Closure|\Illuminate\Database\Query\Builder|string  $table
     * @param  string|null  $as
     * @param  string|null  $connection
     * @return \Illuminate\Database\Query\Builder
     */
    public static function table($table, $as = null, $connection = null)

#### `cursor` 方法

**影響程度：低**

`cursor` 方法現在會回傳 `Illuminate\Support\LazyCollection` 實例來取代 `Generator`。`LazyCollection` 可以像產生器一樣被疊代：

    $users = App\User::cursor();

    foreach ($users as $user) {
        //
    }

<a name="eloquent"></a>
### Eloquent

<a name="belongs-to-update"></a>
#### `BelongsTo::update` 方法

**影響程度：中**

為了一致性，`BelongsTo` 關聯的 `update` 方法現在能被作為更新查詢，這意味著它不再提供批量執行保護或觸發 Eloquent 事件。這樣可以使所有關聯的 `update` 方法用法一致。

如果你想要透過 `BelongsTo` 關聯來更新附加的模型並接收批量執行更新保護與事件，請呼叫模型自身的 `update` 方法：

    // 專用查詢... 沒有批量執行保護或事件...
    $post->user()->update(['foo' => 'bar']);

    // 模型更新... 提供批量執行保護與事件...
    $post->user->update(['foo' => 'bar']);

<a name="eloquent-to-array"></a>
#### 可陣列化與 `toArray`

**影響程度：中**

Eloquent 模型的 `toArray` 方法現在會實作 `Illuminate\Contracts\Support\Arrayable` 來轉換任何屬性成陣列。

<a name="eloquent-primary-key-type"></a>
#### 主鍵類別的宣告

**影響程度：中**

Laravel 6.0 已接受針對數值型別鍵的[性能優化](https://github.com/laravel/framework/pull/28153)。如果你正在使用字串作為模型的主鍵，請在模型上的 `$keyType` 屬性宣告主鍵的型別：

    /**
     * 主鍵 ID 的「型別」。
     *
     * @var string
     */
    protected $keyType = 'string';

### 信件驗證

<a name="email-verification-route"></a>
#### 重寄驗證路由 HTTP 方法

**影響程度：中**

為了預防可能的 CSRF 攻擊，當使用 Laravel 內建的 email 驗證時，路由器註冊的 `email/resend` 路由已從 `GET` 路由更新為 `POST` 路由。因此，你會需要更新前端來發送正確的請求類型到這個路由。例如，如果正在使用內建的 email 驗證模板框架：

    {{ __('繼續之前，請檢查你的 email 的驗證連結。') }}
    {{ __('若你沒收到 email。') }},

    <form class="d-inline" method="POST" action="{{ route('verification.resend') }}">
        @csrf

        <button type="submit" class="btn btn-link p-0 m-0 align-baseline">
            {{ __('點擊這裡來請求其他服務') }}
        </button>.
    </form>

<a name="mustverifyemail-contract"></a>
#### `MustVerifyEmail` Contract

**影響程度：低**

`Illuminate\Contracts\Auth\MustVerifyEmail` Contract 加入了新方法 `getEmailForVerification`。如果你正好有手動實作這個 Contract，請記得實作這個方法。這個方法會回傳要被驗證的 email 位置。如果你的 `App\User` 模型有用到 `Illuminate\Auth\MustVerifyEmail` Trait，那麼無需做修改，因為這個 Trait 已經為你實作這個方法了。

<a name="email-verification-route-change"></a>
#### Email 驗證路由變更

**影響程度：中**

用來驗證 email 的路由路徑已從 `/email/verify/{id}` 改成 `/email/verify/{id}/{hash}`。在升級到 Laravel 6.x 之前寄送的任何 email 驗證的信件會在升級後無效，並顯示 404 頁面。如果有需要，可以定義一組路由來承接舊的驗證路由路徑並為你的使用者顯示一則訊息來告知他們需要重新驗證他們的的 email 位置。

<a name="helpers"></a>
### 輔助函式

#### 陣列與字串的輔助函式套件包

**影響程度：高**

所有 `str_` 與 `array_` 輔助函式已被移到新的 `laravel/helpers` Composer 套件並從框架中移除。如果有需要，你可以將所有呼叫過這些輔助函式更新成使用 `Illuminate\Support\Str` 和 `Illuminate\Support\Arr` 類別。或者，你可以加入 `laravel/helpers` 新套件來繼續使用這些輔助函式：

    composer require laravel/helpers

### 本地化

<a name="trans-and-trans-choice"></a>
#### `Lang::trans` 與 `Lang::transChoice` 方法

**影響程度：中**

翻譯器的 `Lang::trans` 和 `Lang::transChoice` 方法已重新命名為 `Lang::get` 和 `Lang::choice`。

還有，如果你是手動實作 `Illuminate\Contracts\Translation\Translator` contract，請將你的實作的 `trans` 和 `transChoice` 方法更新為 `get` 和 `choice`。

<a name="get-from-json"></a>
#### `Lang::getFromJson` 方法

**影響程度：中**

`Lang::get` 和 `Lang::getFromJson` 方法已被整併。呼叫 `Lang::getFromJson` 方法應改為呼叫 `Lang::get`。

> {note} 請執行 Artisan 的 `php artisan view:clear` 指令來避免 Blade 錯誤的關聯到被移除的 `Lang::transChoice`、`Lang::trans` 和 `Lang::getFromJson`。

### 信件

#### 正式移除 Mandrill 與 SparkPost 驅動

**影響程度：低**

`mandrill` 與 `sparkpost` 郵件驅動已正式被移除了。如果你想要繼續使用這些驅動，我們建議你選擇採用社群維護的套件來提供這個驅動。

### 通知

#### 正式移除 Nexmo 路由

**影響程度：低**

Nexmo 通知頻道的剩餘部分已正式從框架核心中移除了。如果你需要透過 Nexmo 來通知，請遵循[文件的指示](/docs/{{version}}/notifications#routing-sms-notifications)手動將 `routeNotificationForNexmo` 方法時實作到可被通知的模型上。

### 密碼重設

#### 密碼驗證

**影響程度：低**

`PasswordBroker` 不再限制或驗證密碼。密碼驗證已由 `ResetPasswordController` 類別來處理，這會使中間驗證顯得多餘且無法客製化。如果你正在使用內建 `ResetPasswordController` 之外的 `PasswordBroker`（或 `Password` Facade），請在傳入密碼到 Broker 之前，驗證所有密碼。

### 隊列

<a name="queue-retry-limit"></a>
#### 隊列重試次數

**影響程度：中**

在 Laravel 之前的版本，`php artisan queue:work` 指令會不停的重試任務。從 Laravel 6.0 起，這個指令預設只會重試一次任務。如果你想要強制不停重試任務，請將 `0` 傳到 `--tries` 選項：

    php artisan queue:work --tries=0

還有，請確保資料庫有 `failed_jobs` 資料表。你能使用 Artisan 的 `queue:failed-table` 指令來產生這個資料表的遷移檔：

    php artisan queue:failed-table

### 請求

<a name="the-input-facade"></a>
#### `Input` Facade

**影響程度：中**

`Input` Facade，主要是 `Request` Facade 的副本，也正是被移除了。如果你正在使用 `Input::get` 方法，現在應該使用 `Request::input` 方法。所有呼叫 `Input` Facade 都可以簡單地更新為 `Request` Facade。

### 排程

#### `between` 方法

**影響程度：低**

在之前的 Laravel 版本中，排程器的 `between` 方法會有跨日期的混亂行為。像是：

    $schedule->command('list')->between('23:00', '4:00');

對大部分使用者來說，這個方法的預期行為是在 23:00 到 4:00 之間的每分鐘執行一次 `list` 指令。但是，在之前的 Laravel 版本中，排程器會在 4:00 到 23:00 之間每分鐘執行一次 `list` 指令。在 Laravel 6.0，這個行為已被修正。

### Storage

<a name="rackspace-storage-driver"></a>
#### Rackspace Storage 驅動已正式移除

**影響程度：低**

`rackspace` 儲存驅動已正式移除。如果你想要繼續使用 Rackspace 作為儲存驅動來提供，我們建議你選擇採用社群維護的套件來提供這個驅動。

### URL 產生器

#### 路由 URL 的產生及附加參數

在之前的 Laravel 版本中，為路由產生 URL 而傳入相關陣列參數到 `route` 輔助函式或 `URL::route` 方法時會將這些參數作為 URL 值，即使參數值在路由路徑中找不到對應的鍵。從 Laravel 6.0 起，這些值會被改做查詢字串來使用。

    Route::get('/profile/{location}', function ($location = null) {
        //
    })->name('profile');

    // Laravel 5.8: http://example.com/profile/active
    echo route('profile', ['status' => 'active']);

    // Laravel 6.0: http://example.com/profile?status=active
    echo route('profile', ['status' => 'active']);

這個變更也有影響到 `action` 輔助函式與 `URL::action` 方法：

    Route::get('/profile/{id}', 'ProfileController@show');

    // Laravel 5.8: http://example.com/profile/1
    echo action('ProfileController@show', ['profile' => 1]);

    // Laravel 6.0: http://example.com/profile?profile=1
    echo action('ProfileController@show', ['profile' => 1]);

### 驗證

#### FormRequest `validationData` 方法

**影響程度：低**

FormRequest 的 `validationData` 方法已從 `protected` 改成 `public`。如果你正要在實作中覆寫這個方法，請把能見度更新為 `public`。

<a name="miscellaneous"></a>
### 其他

我們也鼓勵你查看 `laravel/laravel` [GitHub repository](https://github.com/laravel/laravel) GitHub 儲存庫中的任何異動。儘管許多更改並不是必要的，但你可能希望保持這些文件與你的應用程序同步。其中一些更改將在本升級指南中介紹，但其他更改（例如更改設定檔案或註釋）將不會被介紹。你可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/5.8...6.x)來輕易的檢查更動的內容，並選擇哪些更新對你比較重要。
