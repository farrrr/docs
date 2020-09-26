# 升級指南

- [從 7.x 升級到 8.0](#upgrade-8.0)

<a name="high-impact-changes"></a>
## 高度影響的變更

<div class="content-list" markdown="1">
- [模型工廠](#model-factories)
- [Queue `retryAfter` 方法](#queue-retry-after-method)
- [Queue `timeoutAt` 屬性](#queue-timeout-at-property)
- [Queue `allOnQueue` 與 `allOnConnection`](#queue-allOnQueue-allOnConnection)
- [預設分頁](#pagination-defaults)
- [Seeder 與 Factory 命名空間](#seeder-factory-namespaces)
</div>

<a name="medium-impact-changes"></a>
## 中度影響的變更

<div class="content-list" markdown="1">
- [需要 PHP 7.3.0](#php-7.3.0-required)
- [失敗任務資料表的批次支援](#failed-jobs-table-batch-support)
- [維護模式的更新]](#maintenance-mode-updates)
- [`php artisan down --message` 選項](#artisan-down-message)
- [`assertExactJson` 方法](#assert-exact-json-method)
</div>

<a name="upgrade-8.0"></a>
## 從 7.x 升級到 8.0

#### 預估升級時間：15 分鐘

> {note} 我們嘗試記錄每個重大變更。由於有些重大的變更是在框架最隱密的地方，但實際上只有一小部分的變更會影響你的應用程式。

<a name="php-7.3.0-required"></a>
### 需要 PHP 7.3.0

**影響程度：中**

PHP 最低支援版本為 7.3.0。

<a name="updating-dependencies"></a>
### 升級依賴項目

在 `composer.json` 檔升級下列的依賴項目。

<div class="content-list" markdown="1">
- `guzzlehttp/guzzle` 更新為 `^7.0.1`
- `facade/ignition` 更新為 `^2.3.6`
- `laravel/framework` 更新為 `^8.0`
- `laravel/ui` 更新為 `^3.0`
- `nunomaduro/collision` 更新為 `^5.0`
- `phpunit/phpunit` 更新為 `^9.0`
</div>

下列為官方套件有支援 Laravel 8 的主要版號。如果有使用到，請在升級之前詳讀各自的升級指南：

<div class="content-list" markdown="1">
- [Horizon v5.0](https://github.com/laravel/horizon/blob/master/UPGRADE.md)
- [Passport v10.0](https://github.com/laravel/passport/blob/master/UPGRADE.md)
- [Socialite v5.0](https://github.com/laravel/socialite/blob/master/UPGRADE.md)
- [Telescope v4.0](https://github.com/laravel/telescope/releases)
</div>

還有，已升級 Laravel 安裝器來支援 `composer create-project` 與 Laravel Jetstream。任何低於 4.0 的安裝器將於 2020 年十月停止服務。請盡快升級全域的安裝器到 `^4.0`。

最後，請檢查你的應用程式所使用的第三方套件的版本是否能支援 Laravel 8。

### 集合

#### `isset` 方法

**影響程度：低**

為了與原生 PHP 行為一致，`Illuminate\Support\Collection` 的 `offsetExists` 方法已更新成使用 `isset`，而不是 `array_key_exists`。當被判斷的集合值為 `null` 時，原本的行為會有變更：

    $collection = collect([null]);

    // Laravel 7.x - true
    isset($collection[0]);

    // Laravel 8.x - false
    isset($collection[0]);

### 資料庫

<a name="seeder-factory-namespaces"></a>
#### Seeder 與 Factory 命名空間

**影響程度：高**

Seeder 與 Factory 現在已有命名空間了。要相容這些變更，請加入 `Database\Seeders` 命名空間到你的 Seeder 類別。還有，之前的 `database/seeds` 目錄需要重新命名為 `database/seeders`：

    <?php

    namespace Database\Seeders;

    use App\Models\User;
    use Illuminate\Database\Seeder;

    class DatabaseSeeder extends Seeder
    {
        /**
         * 填充應用程式的資料庫。
         *
         * @return void
         */
        public function run()
        {
            ...
        }
    }

如果你選擇使用 `laravel/legacy-factories` 套件，就不需要修改 Factory 類別。然而，如果是更新 Factory，請加入 `Database\Factories` 命名空間到那些類別。

接著，請在 `composer.json` 檔案中把 `classmap` 區塊從 `autoload` 的部分中移除，並加入新的命名空間類別目錄的對照表：

    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Database\\Factories\\": "database/factories/",
            "Database\\Seeders\\": "database/seeders/"
        }
    },

### Eloquent

<a name="model-factories"></a>
#### 模型工廠

**影響程度：高**

Laravel 的[模型工廠](/docs/{{version}}/database-testing#creating-factories)功能已完全重寫來支援類別並不再向下相容 Laravel 7.x 之前的 Factory 寫法。不過，為了簡化升級過程，已建立了新的 `laravel/legacy-factories` 套件來繼續與 Laravel 8.x 使用之前的 Factory。請透過 Composer 安裝這個套件：

    composer require laravel/legacy-factories

#### `Castable` 介面

**影響程度：低**

`Castable` 介面的 `castUsing` 方法已更新為接受一組陣列參數。如果你正在實作這個介面，請更新相對應的程式碼實作：

    public static function castUsing(array $arguments);

#### Increment 與 Decrement 事件

**影響程度：低**

在 Eloquent 模型實例上執行 `increment` 或 `decrement` 方法時，現在會正確觸發「update」與「save」關聯模型的事件。

### 事件

#### `Dispatcher` Contract

**影響程度：低**

`Illuminate\Contracts\Events\Dispatcher` Contract 的 `listen` 方法已將 `$listener` 屬性更新為改為非必要。這個改變是為了透過反射來支援自動檢查要處理的事件類型，如果你是手動實作這個介面，請更新相對的程式碼實作：

    public function listen($events, $listener = null);

### 框架

<a name="maintenance-mode-updates"></a>
#### 維護模式的更新

**影響程度：非必要**

Laravel 的[維護模式](/docs/{{version}}/configuration#maintenance-mode)功能已從 Laravel 8.x 中得到改善。現在支援預渲染維護模式模板並除去了末端使用者在維護模式期間會遇到錯誤的機會。但是，要支援這個功能，必須將下列程式碼新增到 `public/index.php` 檔。這些程式碼需要直接放在現有的 `LARAVEL_START` 常數定義下方：

    define('LARAVEL_START', microtime(true));

    if (file_exists(__DIR__.'/../storage/framework/maintenance.php')) {
        require __DIR__.'/../storage/framework/maintenance.php';
    }

<a name="artisan-down-message"></a>
#### `php artisan down --message` 選項

**影響程度：中**

`php artisan down` 指令的 `--message` 選項已正式被移除。或者，請考慮固定訊息來[預渲染維護模式視圖](/docs/{{version}}/configuration#maintenance-mode)。

#### Manager `$app` 屬性

**影響程度：低**

之前不推薦使用的 `Illuminate\Support\Manager` 類別的 `$app` 屬性已正式被移除了。如果你還在用這個屬性，請改用 `container` 屬性。

#### `elixir` 輔助函式

**影響程度：低**

之前不推薦的 `elixir` 輔助函式已正式被移除了。鼓勵還在使用這個方法的應用程式升級到 [Laravel Mix](https://github.com/JeffreyWay/laravel-mix)。

### 郵件

### `sendNow` 方法

**影響程度：低**

先前不推薦使用的 `sendNow` 方法已正式被移除了。請改用 `send` 方法。

### 分頁

<a name="pagination-defaults"></a>
#### 預設分頁

**影響程度：高**

現在分頁器使用 [Tailwind CSS 框架](https://tailwindcss.com)作為它的預設樣式。若要繼續使用，請在 `AppServiceProvider` 的 `boot` 方法中新增下列方法：

    use Illuminate\Pagination\Paginator;

    Paginator::useBootstrap();

### 隊列

<a name="queue-retry-after-method"></a>
#### `retryAfter` 方法

**影響程度：高**

為了與 Laravel 的其他功能有一致性，隊列任務、郵件、通知與監聽器的 `retryAfter` 方法與 `retryAfter` 屬性已重新命名為 `backoff`。請在應用程式的相關類別中更新這個方法與屬性的名稱。

<a name="queue-timeout-at-property"></a>
#### `timeoutAt` 屬性

**影響程度：高**

隊列任務、通知與監聽器的 `timeoutAt` 屬性已重新命名為 `retryUntil`，請在應用程式的相關類別中更新這個方法與屬性的名稱。

<a name="queue-allOnQueue-allOnConnection"></a>
#### `allOnQueue()` / `allOnConnection()` 方法

**影響程度：高**

為了與其他觸發方法的方式一致，被用於串接任務的 `allOnQueue()` 和 `allOnConnection()` 方法已正式被移除了。請改用 `onQueue()` 和 `onConnection()` 方法。這些方法應該在呼叫 `dispatch` 方法之前被呼叫：

    ProcessPodcast::withChain([
        new OptimizePodcast,
        new ReleasePodcast
    ])->onConnection('redis')->onQueue('podcasts')->dispatch();

<a name="failed-jobs-table-batch-support"></a>
#### 失敗任務資料表的批次支援

**影響程度：非必要**

如果你計畫使用 Laravel 8.x 的[任務批次](/docs/{{version}}/queues#job-batching)功能，將需要更新資料庫的 `failed_jobs` 資料表。首先，新增 `uuid` 欄位到資料表中：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('failed_jobs', function (Blueprint $table) {
        $table->string('uuid')->after('id')->nullable()->unique();
    });

接著，在你的 `queue` 設定檔中的 `failed.driver` 設定選項更新為 `database-uuids`。

### 路由

#### 控制器命名空間前綴的自動化

**影響程度：非必要**

之前的 Laravel 版本，`RouteServiceProvider` 有包含值為 `App\Http\Controllers` 的 `$namespace` 屬性。像是在呼叫 `action` 輔助函式的時候，這個屬性值被用來自動前綴控制器路由的宣告與控制器路由的 URL 產生。

在 Laravel 8，這個屬性預設會是 `null`。這可使控制器路由使用標準的 PHP 呼叫語法來宣告，以便支援在許多 IDE 中提供更好的跳轉到控制器類別。

    use App\Http\Controllers\UserController;

    // 使用 PHP 呼叫語法...
    Route::get('/users', [UserController::class, 'index']);

    // 使用字串語法...
    Route::get('/users', 'App\Http\Controllers\UserController@index');

在大多數情況下，這並不會影響升級的應用程式，因為你的 `RouteServiceProvider` 還保有 `$namespace` 屬性與先前的值。然而，如果你是透過建立新的 Laravel 專案來升級應用程式，就有可能遇到重大變更。

如果你想要繼續使用原生的自動前綴控制器路由，請在 `RouteServiceProvider` 中設定 `$namespace` 屬性的值，並在 `boot` 方法中更新路由註冊來使用 `$namespace` 屬性：

    class RouteServiceProvider extends ServiceProvider
    {
        /**
         * 應用程式的「home」 路由的路徑。
         *
         * Laravel 認證會用它來重導剛登入的使用者。
         *
         * @var string
         */
        public const HOME = '/home';

        /**
         * 如果有指定，這個命名空間會自動應用到控制器路由。
         *
         * 還有，這是用來設定 URL 產生器的 root 命名空間。
         *
         * @var string
         */
        protected $namespace = 'App\Http\Controllers';

        /**
         * 定義路由模型綁定、模式、過濾器等。
         *
         * @return void
         */
        public function boot()
        {
            $this->configureRateLimiting();

            $this->routes(function () {
                Route::middleware('web')
                    ->namespace($this->namespace)
                    ->group(base_path('routes/web.php'));

                Route::prefix('api')
                    ->middleware('api')
                    ->namespace($this->namespace)
                    ->group(base_path('routes/api.php'));
            });
        }

        /**
         * 限制應用程式使用頻率。
         *
         * @return void
         */
        protected function configureRateLimiting()
        {
            RateLimiter::for('api', function (Request $request) {
                return Limit::perMinute(60);
            });
        }
    }

### 排程

#### `cron-expression` 函式庫

**影響程度：低**

Laravel 對 `dragonmantank/cron-expression` 的依賴項目已從 `2.x` 升級到 `3.x`。這不會對你的應用程式產生重大的影響，除非你有直接與 `cron-expression` 函式庫互動。如果你直接與這個函式庫互動，請詳讀它的[變更記錄]](https://github.com/dragonmantank/cron-expression/blob/master/CHANGELOG.md)。

### Session

#### `Session` Contract

**影響程度：低**

`Illuminate\Contracts\Session\Session` Contract 已新增了 `pull` 方法。如果你正在手動實作這個 Contract，請更新相對應的實作：

    /**
     * 取得給定鍵的值，然後捨棄它。
     *
     * @param  string  $key
     * @param  mixed  $default
     * @return mixed
     */
    public function pull($key, $default = null);

### 測試

<a name="assert-exact-json-method"></a>
#### `assertExactJson` 方法

**影響程度：中**

`assertExactJson` 方法現在會比較陣列的整數鍵是否有相同的順序。如果你想要比較 JSON 但又不要求陣列整數鍵要有相同的順序，請改用 `assertSimilarJson` 方法。

### 驗證

### 資料庫規則連線

**影響程度：低**

`unique` 與 `exists` 方法現在會在執行查詢時遵循 Eloquent 模型特定的連線名稱（透過模型的 `getConnectionName` 方法來存取）。

<a name="miscellaneous"></a>
### 其他

我們也鼓勵你查看 `laravel/laravel` [GitHub repository](https://github.com/laravel/laravel) GitHub 儲存庫中的任何異動。儘管許多更改並不是必要的，但你可能希望保持這些文件與你的應用程序同步。其中一些更改將在本升級指南中介紹，但其他更改（例如更改設定檔案或註釋）將不會被介紹。你可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/7.x...master)來輕易的檢查更動的內容，並選擇哪些更新對你比較重要。
