# 升級指南

- [從 5.7 升級到 5.8.0](#upgrade-5.8.0)

<a name="high-impact-changes"></a>
## 高度影響的變更

<div class="content-list" markdown="1">
- [快取 TTL 改以秒為單位](#cache-ttl-in-seconds)
- [快取 Lock 的安全性優化](#cache-lock-safety-improvements)
- [解析環境變數](#environment-variable-parsing)
- [Markdown 檔案目錄的變更](#markdown-file-directory-change)
- [Nexmo / Slack 通知頻道](#nexmo-slack-notification-channels)
- [新的預設密碼長度](#new-default-password-length)
</div>

<a name="medium-impact-changes"></a>
## 中度影響的變更

<div class="content-list" markdown="1">
- [產生器與被標籤服務](#container-generators)
- [SQLite 版本限制](#sqlite)
- [比起輔助函式，更傾向於字串與陣列的類別](#string-and-array-helpers)
- [延長服務提供者](#deferred-service-providers)
- [PSR-16 一致性](#psr-16-conformity)
- [模型名稱以不規則複數結尾](#model-names-ending-with-irregular-plurals)
- [自訂中介模型的遞增標號](#custom-pivot-models-with-incrementing-ids)
- [Pheanstalk 4.0](#pheanstalk-4)
- [Carbon 2.0](#carbon-2.0)
</div>

<a name="upgrade-5.8.0"></a>
## 從 5.7 升級到 5.8.0

#### 預估升級時間：1 小時

> {note} 我們嘗試記錄每個重大變更。由於有些重大的變更是在框架最隱密的地方，但實際上只有一小部分的變更會影響你的應用程式。

<a name="updating-dependencies"></a>
### 更新依賴項目

在 `composer.json` 檔案中更新 `laravel/framework` 到 `5.8.*`。

最後，請檢查你的應用程式所使用的第三方套件的版本是否能支援 Laravel 5.8。

<a name="the-application-contract"></a>
### `Application` Contract

#### `environment` 方法

**影響程度：非常低**

`Illuminate\Contracts\Foundation\Application` Contract 的 `environment` 方法[已有些改變](https://github.com/laravel/framework/pull/26296)。如果你有實作這個 Contract，請更新該方法：

    /**
     * 取得或檢查當前應用程式的環境。
     *
     * @param  string|array  $environments
     * @return string|bool
     */
    public function environment(...$environments);

#### 新增方法

**影響程度：非常低**

[`Illuminate\Contracts\Foundation\Application` contract](https://github.com/laravel/framework/pull/26477) 已加入的方法有 `bootstrapPath`、`configPath`、`databasePath`、`environmentPath`、`resourcePath`、`storagePath`、`resolveProvider`、`bootstrapWith`、`configurationIsCached`、`detectEnvironment`、`environmentFile`、`environmentFilePath`、`getCachedConfigPath`、`getCachedRoutesPath`、`getLocale`、`getNamespace`、`getProviders`、`hasBeenBootstrapped`、`loadDeferredProviders`、`loadEnvironmentFrom`、`routesAreCached`、`setLocale`、`shouldSkipMiddleware` 和 `terminate` 方法。

如果你正在實作這個介面，請將這些方法加到你的實作中。

<a name="authentication"></a>
### 認證

#### 密碼重設通知的路由參數

**影響程度：低**

當使用者請求一組用來重設密碼的連結時，Laravel 會使用 `route` 輔助方法來產生 URL，該路由會以 `password.reset` 來命名並建立。若是使用 Laravel 5.7，Token 要被傳入沒有指定鍵名的 `route` 輔助函式，例如：

    route('password.reset', $token);

若是使用 Laravel 5.8，Token 要被傳入指定鍵名參數的 `route` 輔助函式：

    route('password.reset', ['token' => $token]);

所以，如果你正在定義自己的 `password.reset` 路由，請確認該 URI 是否有包含 `{token}` 參數。

<a name="new-default-password-length"></a>
#### 新的預設密碼長度

**影響程度：高**

選擇或重設密碼所需的密碼長度[已改為八個字元](https://github.com/laravel/framework/pull/25957)。請更新在應用程式中任何驗證規則或邏輯來符合這個新預設的八個字元。

若需要保留之前六個字元的長度或其他長度，可以繼承 `Illuminate\Auth\Passwords\PasswordBroker` 類別並覆寫 `validatePasswordWithDefaults` 方法的自訂邏輯。

<a name="cache"></a>
### 快取

<a name="cache-ttl-in-seconds"></a>
#### TTL 改以秒為單位

**影響程度：非常高**

為了在儲存項目時能有更細節的有效時間設定，快取項目的有效時間已從分鐘改為秒鐘。這已更新了 `Illuminate\Cache\Repository` 類別與繼承的類別的 `put`、`putMany`、`add`、`remember` 和 `setDefaultCacheTime` 方法，以及每個儲存快取的 `put` 方法。請詳閱[這個 PR](https://github.com/laravel/framework/pull/27276) 來獲得更多資訊。

如果有傳遞數值給這些方法，請更新程式碼來確保現在傳遞的數字是預期要保留在快取的秒數。此外，你可以傳遞 `DateTime` 實例來設定該項目的有效時間：

    // Laravel 5.7 - 儲存項目 30 分鐘...
    Cache::put('foo', 'bar', 30);

    // Laravel 5.8 - 儲存項目 30 秒鐘...
    Cache::put('foo', 'bar', 30);

    // Laravel 5.7 / 5.8 - 儲存項目 30 秒鐘...
    Cache::put('foo', 'bar', now()->addSeconds(30));

> {tip} 這個變更讓 Laravel 快取系統完全符合於 [PSR-16 快取函式庫標準](https://www.php-fig.org/psr/psr-16/)。

<a name="psr-16-conformity"></a>
#### PSR-16 一致性

**影響程度：中**

除了[後面會提到的回傳值變更](#the-repository-and-store-contracts)外，`Illuminate\Cache\Repository` 類別的 `put`、`putMany` 和 `add` 方法的 TTL 參數也更新成 PSR-16 標準規範。新的用法會提供 `null` 預設值，所以不指定 TTL 就會導致永久儲存項目的快取。還有，將儲存 TTL 設為 0 或更低的快取項目會被從快取中移除。請詳閱[這個 PR](https://github.com/laravel/framework/pull/27265) 來獲得更多資訊。

這些變更[也更新了](https://github.com/laravel/framework/pull/27265) `KeyWritten` 事件。

<a name="cache-lock-safety-improvements"></a>
#### Lock 的安全性優化

**影響程度：高**

在 Laravel 5.7 與更之前的 Laravel 版本，某些快取驅動提供的「原子鎖」功能會在發生非預期的行為下導致提早釋出鎖定。

例如：**使用者 A** 取得 10 秒有效時間來鎖定 `foo`。**使用者 A** 實際上花費了 20 秒才完成任務。快取系統會在**使用者 A** 處理執行的十秒後自動釋放鎖定。**使用者 B** 取得了鎖定的 `foo`。**使用者 A** 最後完成了任務並釋放了 `foo` 的鎖定，這也不小心的把**使用者 B** 的鎖定給釋放掉。此時**使用者 C** 能夠取得該鎖定。

為了降低這個情況發生，現在 Laravel 會加入「Scope Token」來產生鎖定。在正常的情況下，只有正確的鎖定擁有者才能釋放該鎖定。

如果你正在使用 `Cache::lock()->get(Closure)` 方法來鎖定，則不需要修改：

    Cache::lock('foo', 10)->get(function () {
        // 鎖會被自動安全的釋放...
    });

然而，如果你是手動呼叫 `Cache::lock()->release()`，必須更新程式碼來維護鎖的實例。接著，在完成任務之後，可以在**同個鎖的實力**上呼叫 `release` 方法。例如：

    if (($lock = Cache::lock('foo', 10))->get()) {
        // 執行任務...

        $lock->release();
    }

有時你想要在一個進程中取得鎖定，並在另一個進程中釋放它。例如，在網頁請求期間取得鎖，並想要在該請求觸發的隊列任務結束時釋放該鎖定。在這個情況下，可以傳入該鎖的「擁有者 Token」到隊列的任務中，以便任務可以使用給定的 Token 來重新實例化鎖定：

    // 在控制器中...
    $podcast = Podcast::find(1);

    if (($lock = Cache::lock('foo', 120))->get()) {
        ProcessPodcast::dispatch($podcast, $lock->owner());
    }

    // 在 ProcessPodcast 任務中...
    Cache::restoreLock('foo', $this->owner)->release();

如果你想要釋放非當前鎖定的擁有者的鎖，可以使用 `forceRelease` 方法

    Cache::lock('foo')->forceRelease();

<a name="the-repository-and-store-contracts"></a>
#### `Repository` 與 `Store` Contract

**影響程度：非常低**

為了符合 `PSR-16`，`Illuminate\Contracts\Cache\Repository` Contract 的 `put` 和 `forever` 方法和 `Illuminate\Contracts\Cache\Store` Contract 的 `putMany` 和 `forever` 方法的回傳值已從 `void` [改為](https://github.com/laravel/framework/pull/26726) `bool`。

<a name="carbon-2.0"></a>
### Carbon 2.0

**影響程度：中**

Laravel 現在同時支援 Carbon 1 與 Carbon 2。所以，Composer 會在沒有與其他任意套件有相容性問題的情況下嘗試升級到 Carbon 2.0。請詳閱[Carbon 2.0 升級指南](https://carbon.nesbot.com/docs/#api-carbon-2)。

<a name="collections"></a>
### 集合

#### `add` 方法

**影響程度：非常低**

`add` 方法已從 Eloquent Collection 類別中[被移到](https://github.com/laravel/framework/pull/27082)基本的 Collection 類別。如果正在擴充 `Illuminate\Support\Collection` 且用到 `add` 方法，請確保該方法參數與它的上層相同：

    public function add($item);

#### `firstWhere` 方法

**影響程度：非常低**

`firstWhere` 方法參數[已改成](https://github.com/laravel/framework/pull/26261)與 `where` 方法的參數相同。如果你正在覆寫這個方法，請更新該方法參數與它的上層相同：

    /**
     * 依照給定的鍵值對來取得第一個項目。
     *
     * @param  string  $key
     * @param  mixed  $operator
     * @param  mixed  $value
     * @return mixed
     */
    public function firstWhere($key, $operator = null, $value = null);

<a name="console"></a>
### 指令

#### `Kernel` Contract

**影響程度：非常低**

[`Illuminate\Contracts\Console\Kernel` contract](https://github.com/laravel/framework/pull/26393) 已加入 `terminate` 方法。如果你正在實作這個介面，請加入這個方法到你的實作中。

<a name="container"></a>
### 容器

<a name="container-generators"></a>
#### 產生器與被標籤服務

**影響程度：中**

容器的 `tagged` 方法現在使用 PHP 產生器來事後實例化給定標籤的服務。這會在你沒用到每個被標籤的服務時提高效能。

因為這個變更，`tagged` 方法現在會回傳 `iterable` 來取代 `array`。如果型別提示這個方法的回傳值，請確保型別提示有被修改為 `iterable`。

還有，無法再透過陣列的位移值來直接存取被標籤的服務，像是 `$container->tagged('foo')[0]`。

#### `resolve` 方法

**影響程度：非常低**

`resolve` 方法[現在可以接受](https://github.com/laravel/framework/pull/27066)新的布林值參數，該參數會指示物件在實例化期間是否執行或提升事件（解析回呼）。如果你正在覆寫這個方法，請更新該方法的參數來符合上一層的實作。

#### `addContextualBinding` 方法

**影響程度：非常低**

[`Illuminate\Contracts\Container\Container` contract](https://github.com/laravel/framework/pull/26551) 已加入 `addContextualBinding` 方法。如果你正在實作這個介面，請加入這個方法到你的實作中。

#### `tagged` 方法

**影響程度：低**

`tagged` 方法參數[已有所變更](https://github.com/laravel/framework/pull/26953)，現在會回傳 `iterable` 而不是 `array`。如果你有在程式碼中取得這個方法的同個參數的回傳值使用 `array` 型別提示，請將型別提示修改為 `iterable`。 

#### `flush` 方法

**影響程度：非常低**

[`Illuminate\Contracts\Container\Container` contract](https://github.com/laravel/framework/pull/26477) 已加入 `flush` 方法。如果你正在實作這個介面，請加入這個方法到你的實作中。

<a name="database"></a>
### 資料庫

#### MySQL JSON 值不會再多加引號

**影響程度：低**

現在使用 MySQL 和 MariaDB 時，查詢建構器會回傳未加引號的 JSON 值。這個用法與其他支援的資料庫一致：

    $value = DB::table('users')->value('options->language');

    dump($value);

    // Laravel 5.7...
    '"en"'

    // Laravel 5.8...
    'en'

結果是不再支援或需要 `->>` 運算子。

<a name="sqlite"></a>
#### SQLite

**影響程度：中**

從 Laravel 5.8 起，[SQLite 支援最遠的版本]（https://github.com/laravel/framework/pull/25995）是 SQLite 3.7.11。如果你使用的是較舊的 SQLite 版本，請更新它（推薦使用 SQLite 3.8.8+）。

#### Migrations & `bigIncrements`

**影響程度：沒有**

[從 Laravel 5.8 開始](https://github.com/laravel/framework/pull/26472)，Migration 範本在預設的 ID 欄位上使用了 `bigIncrements` 方法。之前，ID 欄位是使用 `increments` 方法來建立。

這不會影響專案中任何現有的程式碼。然而，請注意外鍵欄位必須為是相同型別。所以使用 `increments` 方法建立的欄位無法參照使用 `bigIncrements` 方法建立的欄位。

<a name="eloquent"></a>
### Eloquent

<a name="model-names-ending-with-irregular-plurals"></a>
#### 模型名稱以不規則複數結尾

**影響程度：中**

從 Laravel 5.8 開始，多字的模型名稱結尾的不規則複數單字將能正確表達複數[將能正確表達複數](https://github.com/laravel/framework/pull/26421)。

    // Laravel 5.7...
    App\Feedback.php -> feedback (correctly pluralized)
    App\UserFeedback.php -> user_feedbacks (incorrectly pluralized)

    // Laravel 5.8
    App\Feedback.php -> feedback (correctly pluralized)
    App\UserFeedback.php -> user_feedback (correctly pluralized)

如果你的模型剛好是錯誤的複數，可以透過定義在模型上的 `$table` 屬性來延用舊的資料表名稱：

    /**
     * 模型對應的的資料表。
     *
     * @var string
     */
    protected $table = 'user_feedbacks';

<a name="custom-pivot-models-with-incrementing-ids"></a>
#### 自訂中介模型的遞增編號

如果你有已經定義了使用自訂中介模型的多對多關聯，且中介模型具備自動遞增的主鍵，請確保自訂中介模型類別有把 `incrementing` 屬性設定為 `true`：

    /**
     * 設定編號是否要自動遞增。
     *
     * @var bool
     */
    public $incrementing = true;

#### `loadCount` 方法

**影響程度：低**

`Illuminate\Database\Eloquent\Model` 類別已加入了 `loadCount` 方法。如果你也有定義 `loadCount` 方法，那麼會與 Eloquent 的定義有衝突。

#### `originalIsEquivalent` 方法

**影響程度：非常低**

`Illuminate\Database\Eloquent\Concerns\HasAttributes` trait 的 `originalIsEquivalent` 方法已從 `protected` [改為](https://github.com/laravel/framework/pull/26391) `public`。

#### 自行轉換軟刪除的 `deleted_at` 屬性

**影響程度：低**

當你的 Eloquent 模型使用 `Illuminate\Database\Eloquent\SoftDeletes` trait 時候，`deleted_at` 屬性[將會自行轉換成](https://github.com/laravel/framework/pull/26985) `Carbon` 實例。你能撰寫自己自訂的存取器屬性或手動加入它到 `casts` 屬性來覆寫這個用法：

    protected $casts = ['deleted_at' => 'string'];

#### BelongsTo `getForeignKey` 與 `getOwnerKey` 方法

**影響程度：低**

`BelongsTo` 關聯的 `getForeignKey`、`getQualifiedForeignKey` 和 `getOwnerKey` 方法已個別重新命名為 `getForeignKeyName`、`getQualifiedForeignKeyName` 和 `getOwnerKeyName`。讓方法名稱與 Laravel 提供的其他關聯一致。

<a name="environment-variable-parsing"></a>
### 解析環境變數

**影響程度：高**

被用來解析 `.env` 檔案的 [phpdotenv](https://github.com/vlucas/phpdotenv) 套件已發佈新的主要版本，這會影響從 `env` 輔助函式回傳的結果。更具體的說，現在未加引號的 `#` 字元會被當作是註解，而不是部分的值：

之前的用法：

    ENV_VALUE=foo#bar

    env('ENV_VALUE'); // foo#bar

新的用法：

    ENV_VALUE=foo#bar
    env('ENV_VALUE'); // foo

要延用過去的用法，請將環境值包在引號中：

    ENV_VALUE="foo#bar"

    env('ENV_VALUE'); // foo#bar

想知道更多資訊，請詳閱 [phpdotenv 升級指南](https://github.com/vlucas/phpdotenv/blob/master/UPGRADING.md)。

<a name="events"></a>
### 事件

#### `fire` 方法

**影響程度：低**

[已移除](https://github.com/laravel/framework/pull/26392) `Illuminate\Events\Dispatcher` 類別的 `fire` 方法（已在 Laravel 5.4 棄用）。
請改用 `dispatch` 方法。

<a name="exception-handling"></a>
### 異常處理

#### `ExceptionHandler` Contract

**影響程度：低**

[`Illuminate\Contracts\Debug\ExceptionHandler` contract](https://github.com/laravel/framework/pull/26193) 已加入 `shouldReport` 方法。如果你正在實作這個介面，請加入這個方法到你的實作中。

#### `renderHttpException` 方法

**影響程度：低**

`Illuminate\Foundation\Exceptions\Handler` 類別的 `renderHttpException` 方法參數[已有變更](https://github.com/laravel/framework/pull/25975)。如果你正在覆寫異常處理器中的這個方法，請對照他的上層用法來更新方法參數：

    /**
     * 渲染給定的 HttpException。
     *
     * @param  \Symfony\Component\HttpKernel\Exception\HttpExceptionInterface  $e
     * @return \Symfony\Component\HttpFoundation\Response
     */
    protected function renderHttpException(HttpExceptionInterface $e);

<a name="mail"></a>
### 郵件

<a name="markdown-file-directory-change"></a>
<a name="markdown-file-directory-change"></a>
### Markdown 檔案目錄變更

**影響程度：高**

如果你有使用 `vendor:publish` 指令來發布 Laravel 的 Markdown 信件模組，請將 `/resources/views/vendor/mail/markdown` 目錄重新命名為 `/resources/views/vendor/mail/text`。

還有，`markdownComponentPaths` 方法[已重新命名為](https://github.com/laravel/framework/pull/26938) `textComponentPaths`。如果你正在覆寫這個方法，請對照它的上層寫法來更新該方法名稱。

#### `PendingMail` 類別的方法參數變更

**影響程度：非常低**

`Illuminate\Mail\PendingMail` 類別的 `send`、`sendNow`、`queue`、`later` 和 `fill` 方法[已更改為](https://github.com/laravel/framework/pull/26790)允許 `Illuminate\Contracts\Mail\Mailable` 實例來取代 `Illuminate\Mail\Mailable`。如果你正在覆寫當中的某些方法，請對照它的上層寫法來更新他們的參數。

<a name="queue"></a>
### 隊列

<a name="pheanstalk-4"></a>
#### Pheanstalk 4.0

**影響程度：中**

Laravel 5.8 提供支援 Pheanstalk 隊列函式庫的 `~4.0` 版本。如果你正在使用 Pheanstalk 函式庫，請透過 Composer 來升級函式庫到 `~4.0` 版本。

#### `Job` Contract

**影響程度：非常低**

`isReleased`、`hasFailed` 和 `markAsFailed` 方法[已加到 `Illuminate\Contracts\Queue\Job` contract](https://github.com/laravel/framework/pull/26908)。如果正在實作這個介面，請加入加入這些方法到你的實作。

#### `Job::failed` 與 `FailingJob` 類別

**影響程度：非常低**

當隊列錯誤發生在 Laravel 5.7 時，隊列器會去執行 `FailingJob::handle` 方法。在 Laravel 5.8，`FailingJob` 類別所具備的邏輯已轉移至各自的任務類別上的 `fail` 方法。也因為如此，`fail` 方法已被加到 `Illuminate\Contracts\Queue\Job` Contract。

`Illuminate\Queue\Jobs\Job` 類別包含了 `fail` 的實作且不需要更動原本的程式碼。然而，如果你正在採用任務類別來建構自訂隊列驅動，且該類別沒有繼承 Laravel 提供的基本任務類別，則需要在自訂任務類別中手動實作 `fail` 方法。可以參考 Laravel 的基本任務類別作為實作的依據。

這個改可以讓自訂隊列驅動更好的控制任務刪除線程。

#### Redis 阻塞彈出

**影響程度：非常低**

現在能安全地使用 Redis 隊列驅動的「阻塞彈出」功能。在過去，Redis 伺服器或執行器如果在檢索資料同時崩潰時，隊列任務會有小機率的掉資料。為了要安全的阻塞彈出，會為每個 Laravel 隊列建立一組附有 `:notify` 後綴的新 Redis 列表。

<a name="requests"></a>
### 請求

#### `TransformsRequest` 中介層

**影響程度：低**

`Illuminate\Foundation\Http\Middleware\TransformsRequest` 中介層的 `transform` 方法現在會在輸入是一組陣列時接收「全部」的請求輸入鍵：

    'employee' => [
        'name' => 'Taylor Otwell',
    ],

    /**
     * 轉換給定的值。
     *
     * @param  string  $key
     * @param  mixed  $value
     * @return mixed
     */
    protected function transform($key, $value)
    {
        dump($key); // 'employee.name' (Laravel 5.8)
        dump($key); // 'name' (Laravel 5.7)
    }

<a name="routing"></a>
### 路由

#### `UrlGenerator` Contract

**影響程度：非常低**

[`Illuminate\Contracts\Routing\UrlGenerator` contract](https://github.com/laravel/framework/pull/25616) 已加入 `previous` 方法。如果你正在實作這個介面，請加入這個方法到你的實作中。

#### `Illuminate\Routing\UrlGenerator` 的 `cachedSchema` 屬性

**影響程度：非常低**

`Illuminate\Routing\UrlGenerator` 的 `$cachedSchema` 屬性名稱（已在 Laravel 5.7 棄用）[已改為](https://github.com/laravel/framework/pull/26728) `$cachedScheme`。

<a name="sessions"></a>
### Sessions

#### `StartSession` 中介層

**影響程度：非常低**

session 延續相關的邏輯已從 [`terminate()` 方法被移到 `handle()` 方法](https://github.com/laravel/framework/pull/26410)。如果你正在覆寫這些方法，請對照這些變更來更新它們。

<a name="support"></a>
### 輔助函式

<a name="string-and-array-helpers"></a>
#### 比起輔助函式，更傾向於字串與陣列的類別

**影響程度：中**

所有 `array_*` 和 `str_*` 全域輔助函式[都已棄用](https://github.com/laravel/framework/pull/26898)。請直接使用 `Illuminate\Support\Arr` 和 `Illuminate\Support\Str` 方法。

由於輔助函式已搬到 [laravel/helpers](https://github.com/laravel/helpers) 套件，這個套件包含所有陣列與字串的全域輔助函式，所以這個變更的影響程度評估為「中度」。

<a name="deferred-service-providers"></a>
#### 延長服務提供者

**影響程度：中**

[已不推薦](https://github.com/laravel/framework/pull/27067)在服務提供者上使用 `defer` 布林屬性來指示提供者是否被延長。為了要標記要被延長的服務提供者，請實作 `Illuminate\Contracts\Support\DeferrableProvider` Contract。

#### `env` 輔助函式只能讀取

**影響程度：低**

在過去，`env` 輔助函式可以在執行時改變從環境變數中取得的值。在 Laravel 5.8 中，`env` 輔助函式會把環境變數視為不可變動的。如果想要在執行時改變環境變數，請考慮使用 `config` 輔助函式取得設定的值：

之前的用法：

    dump(env('APP_ENV')); // local

    putenv('APP_ENV=staging');

    dump(env('APP_ENV')); // staging

新的用法：

    dump(env('APP_ENV')); // local

    putenv('APP_ENV=staging');

    dump(env('APP_ENV')); // local

<a name="testing"></a>
### 測試

#### `setUp` 與 `tearDown` 方法

`setUp` 和 `tearDown` 方法現在需要 void 回傳類別。

    protected function setUp(): void
    protected function tearDown(): void

#### PHPUnit 8

**影響程度：非必要**

Laravel 5.8 預設是使用 PHPUnit 7。不過，你可以選擇升級到 PHPUnit 8，但這需要 PHP >= 7.2。另外，請詳閱[PHPUnit 8 發布的公告](https://phpunit.de/announcements/phpunit-8.html)的全部變更列表。

<a name="validation"></a>
### 驗證

#### `Validator` Contract

**影響程度：非常低**

[`Illuminate\Contracts\Validation\Validator` contract](https://github.com/laravel/framework/pull/26419) 已加入 `validated` 方法：

    /**
     * 取得以驗證的屬性與值。
     *
     * @return array
     */
    public function validated();

如果你正在實作這個介面，請加入這個方法到你的實作中。

#### `ValidatesAttributes` Trait

**影響程度：非常低**

`Illuminate\Validation\Concerns\ValidatesAttributes` trait 的 `parseTable`、`getQueryColumn` 和 `requireParameterCount` 方法已從 `protected` 改為 `public`。

#### `DatabasePresenceVerifier` 類別

**影響程度：非常低**

`Illuminate\Validation\DatabasePresenceVerifier` 類別的 `table` 方法已從 `protected` 改為 `public`。

#### `Validator` 類別

**影響程度：非常低**

`Illuminate\Validation\Validator` 類別的 `getPresenceVerifierFor` 方法已從 `protected` [改為](https://github.com/laravel/framework/pull/26717) `public`。

#### Email 驗證

**影響程度：非常低**

現在 Email 驗證規則會檢查 Email 是否符合 [RFC6530](https://tools.ietf.org/html/rfc6530)，藉此讓驗證邏輯與 SwiftMailer 所用的邏輯一致。在 Laravel `5.7`，`email` 規則只驗證 Email 是否符合 [RFC822](https://tools.ietf.org/html/rfc822)。

因此，當在使用 Laravel 5.8 時，過去被誤判為無效的 Email 現在都將會是有效的（例如：`hej@bär.se`）。理論上，這應該視為漏洞修復。然而，出於謹慎考量，將它列為重大變更。[如果你有遇到與此變更有關的任何問題，請通知我們](https://github.com/laravel/framework/pull/26503)。

<a name="view"></a>
### 視圖

#### `getData` 方法

**影響程度：非常低**

 [`Illuminate\Contracts\View\View` contract](https://github.com/laravel/framework/pull/26754) 新增了 `getData` 方法。如果你正在實作這個介面，請新增這個方法到你的實作中。

<a name="notifications"></a>
### 通知

<a name="nexmo-slack-notification-channels"></a>
#### Nexmo / Slack 通知頻道

**影響程度：高**

Nexmo 與 Slack 通知頻道已被移出首要套件。若要使用這些頻道，需要引入下列套件：

    composer require laravel/nexmo-notification-channel
    composer require laravel/slack-notification-channel

<a name="miscellaneous"></a>
### 其他

我們也鼓勵你查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel)中的任何異動。儘管許多更改並不是必要的，但你可能希望保持這些文件與你的應用程序同步。其中一些更改將在本升級指南中介紹，但其他更改（例如更改設定檔案或註釋）將不會被介紹。你可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/5.7...5.8)來輕易的檢查更動的內容，並選擇哪些更新對你比較重要。