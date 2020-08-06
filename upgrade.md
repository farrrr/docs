# 升級指南

- [從 5.6 升級到 5.6.30](#upgrade-5.6.30)
- [從 5.5 升級到 5.6.0](#upgrade-5.6.0)

<a name="upgrade-5.6.30"></a>
## 從 5.6 升級到 5.6.30（安全性更新版本）

Laravel 5.6.30 是 Laravel 安全性更新版本，建議所有使用者即刻升級。Laravel 5.6.30 也有對 Cookie 加密與序列化邏輯的重大變更，所以在升級應用程式前請詳閱以下說明。

**這個漏洞只會在惡意使用者持有了你的應用程式的加密金鑰（`APP_KEY` 環境變數）時發生。**一般情況下，你的使用者是不可能存取到這個值。然而，有權限存取加密金鑰的前員工就有機會使用這個金鑰來攻擊你的應用程式。如果你有任何理由來讓你相信自身的加密金鑰已落入不肖人士手裡，則**一定要**翻新這個金鑰的值。

### Cookie 序列化

Laravel 5.6.30 已經停用所有 Cookie 值的序列化與反序列化。由於所有 Laravel Cookie 都有被加密與簽章的，所以理論上 Cookie 的值是可以防止客戶端的竄改的。**然而，如果你的應用程式的加密金鑰落入不肖人士手中，則他們可以使用該加密金鑰來製作 Cookie 值，並利用 PHP 物件的序列化與反序列化的既有漏洞，像是在應用程式內呼叫任意類別的方法。**

停用所有 Cookie 值的序列化會使你的應用程式的所有 Session 無效，以及所有使用者會需要再次登入應用程式（如果他們有用到 `remember_token`，這時使用者會自動被登入）。還有，你的應用程式當時正在加密 Cookie 任何值都會無效喔。也因為這樣，你可能希望加入額外的邏輯到應用程式來驗證自訂的 Cookie 值是否與預期的值清單相符；如果沒有相符，你應該捨棄他們。

#### 設定 Cookie 序列化

因為無法存取你應用程式的加密金鑰，所以這個漏洞無法被使用。我們選擇了一種做法來提供你在重新啟用加密的 Cookie 序列化時也能讓應用程式相容這些變更。要啟用與停用 Cookie 序列化，可以更改 `App\Http\Middleware\EncryptCookies` [中介層](https://github.com/laravel/laravel/blob/5.6/app/Http/Middleware/EncryptCookies.php)的 `serialize` 靜態屬性：

    /**
     * 是否要序列化 Cookie。
     *
     * @var bool
     */
    protected static $serialize = true;

> **注意：** 當你啟用加密的 Cookie 序列化後，若應用程式的加密金鑰落入不肖人士手中，就容易遭受攻擊。如果你認為你的金鑰落入不肖人士手中，你應該在啟用加密 Cookie 序列化之前翻新金鑰的值。

### Dusk 4.0.0

已發布 Dusk 4.0.0，且不會序列化 Cookie。如果你選擇啟用 Cookie 序列化，你應該繼續使用 Dusk 3.0.0。不然，你應該升級到 4.0.0。

### Passport 6.0.7

已發布 Passport 6.0.7，並加入 `Laravel\Passport\Passport::withoutCookieSerialization()` 新方法。在你停用 Cookie 序列化後，你可以在應用程式的 `AppServiceProvider` 中呼叫這個方法。

<a name="upgrade-5.6.0"></a>
## 從 5.5 升級到 5.6.0

#### 預估升級時間：10 - 30 分鐘

> {note} 我們嘗試記錄每個重大變更。由於有些重大的變更是在框架最隱密的地方，但實際上只有一小部分的變更會影響你的應用程式。

### PHP

Laravel 5.6 需要 PHP 7.1.3 或更高的版本。

### 更新依賴項目

在 `composer.json` 檔案中更新 `laravel/framework` 到 `5.6.*` 還有 `fideloper/proxy` 更新到 `^4.0`。

如果你有使用 `laravel/browser-kit-testing` 套件，則要在 composer.json 更新該套件到 `4.*`。

還有，如果你有使用第三方的 Laravel 套件，你應該升級他們到他們最後發布的版本：

<div class="content-list" markdown="1">
- Dusk（升級到 `^3.0`）
- Passport（升級到 `^6.0`）
- Scout（升級到 `^4.0`）
</div>

最後，請檢查你的應用程式所使用的第三方套件的版本是否能支援 Laravel 5.6。

#### Symfony 4

Laravel 所使用的底層 Symfony 元件都已升級到 `^4.0` 發布的版本。如果你有在應用程式中直接使用 Symfony 元件，請你先看 [Symfony 更新記錄](https://github.com/symfony/symfony/blob/master/UPGRADE-4.0.md)。

#### PHPUnit

你應該升級 `phpunit/phpunit` 到 `^7.0`。

### 陣列

#### `Arr::wrap` 方法

現在傳入 `null` 到 `Arr::warp` 方法會回傳一組空陣列。

### Artisan

#### `optimize` 命令

之前棄用的 `optimize` Artisan 命令已被移除。隨著近年來 PHP 自身的 OPcache 的改良，`optimize` 命令不再提供任何相關的性能優化。因此，你可以從 `composer.json` 檔案移除 `scripts` 中的 `php artisan optimize`。

### Blade

#### HTML 實體編碼

在之前的 Laravel 版本中，Blade（與 `e` 輔助函式）不會對 HTML 實體進行二次編碼。這並不是底層 `htmlspecialchars` 底層預設的用法，且
在渲染內容或直接傳遞 JSON 內容到 JaveScript 框架時可能會導致意外發生。

在 Laravel 5.6，Blade 與 `e` 輔助函式預設會對特殊字元做二次編碼。讓這些功能與預設 PHP 底層的 `htmlspecialchars` 函式的用法保持一致性。如果你想要維持過去防止防止二次編碼的用法，可以使用 `Blade::withoutDoubleEncoding`：

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Blade;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 啟動任何應用程式服務。
         *
         * @return void
         */
        public function boot()
        {
            Blade::withoutDoubleEncoding();
        }
    }

### 快取

#### 限制使用率 `tooManyAttempts` 方法

已經從這個方法中移除未被使用 `$decayMinutes` 參數。如果你有自己的實例覆寫過這個方法，也請從這個方法中刪除該參數。

### 資料庫

#### 多型欄位的索引排序

為了提高性能，將對調建構 `morphs` 遷移方法的欄位索引。如果你有在 Migrate 中用過 `morphs` 方法，則會在執行 Migrate 的 `down` 方法時收到錯誤訊息。如果應用程式還在開發階段，你可以使用 `migrate:fresh` 命令來重頭開始重建資料庫。如果應用程式已正式上線，則要把索引名稱傳遞給 `morphs` 方法。

#### 新增 `MigrationRepositoryInterface` 方法

新的 `getMigrationsBatches` 方法已新增到 `MigrationRepositoryInterface`。你可能在極少數的情況下會定義這個類別到自己的實例上，請將這個方法加到你的實例中。你能以框架為例來查看預設的實例。

### Eloquent

#### `getDateFormat` 方法

`getDateFormat` 方法現在是 `public`，而不是 `protected`。

### 雜湊化

#### 新的設定檔

現在所有的雜湊設定都放在 `config/hashing.php` 設定檔。你可以在自己的應用程式放置[預設設定檔](https://github.com/laravel/laravel/blob/5.6/config/hashing.php)的副本。你想必會將 `bcrypt` 驅動繼續做為預設的驅動。不過，也有支援 `argon`。

### 輔助函式

#### `e` 輔助方法

在之前的 Laravel 版本中，Blade（與 `e` 輔助函式）不會對 HTML 實體進行二次編碼。這並不是底層 `htmlspecialchars` 底層預設的用法，且
在渲染內容或直接傳遞 JSON 內容到 JaveScript 框架時可能會導致意外發生。

在 Laravel 5.6，Blade 與 `e` 輔助函式預設會對特殊字元做二次編碼。讓這些功能與預設 PHP 底層的 `htmlspecialchars` 函式的用法保持一致性。果你想要維持過去防止防止二次編碼的用法，可以傳入 `false` 到 `e` 輔助函式的第二個參數：

    <?php echo e($string, false); ?>

### 日誌系統

#### 新的設定檔

現在所有日誌設定全都放置在 `config/logging.php` 設定檔。你可以在自己的應用程式放置[預設設定檔](https://github.com/laravel/laravel/blob/5.6/config/logging.php)的副本，並依據應用程式的需求來調整設定。

可以從 `config/app.php` 設定檔中刪除 `log` 和 `log_level` 設定選項。

#### `configureMonologUsing` 方法

如果你有用到 `configureMonologUsing` 方法來為你的應用程式客製化 Monolog 實例。你現在能建立 `custom` 日誌頻道。更多關於如何建立自訂頻道，請詳閱[完整日誌文件](/docs/5.6/logging#creating-custom-channels)。

#### 日誌 `Writer` 類別

`Illuminate\Log\Writer` 類別已重新命名為 `Illuminate\Log\Logger`。如果你有使用這個類別作為應用程式的依賴項目之一，應該更新該類別到新名字。或者，你可以考慮使用標準化的 `Psr\Log\LoggerInterface` 來取代之。

#### `Illuminate\Contracts\Logging\Log` 介面

由於這個介面完全重複 `Psr\Log\LoggerInterface` 介面，所以已移除這個介面。你應該使用 `Psr\Log\LoggerInterface` 介面來取代。

### 郵件

#### `withSwiftMessage` 回呼

在 Laravel 之前的版本，在內容準備好編碼並加到訊息_之後_，才使用 `withSwiftMessage` 來註冊 Swift 訊息客製化回呼。現在這些回呼會在內容被加入_之前_呼叫，這能讓你根據需求來客製化編碼或其他訊息選項。

### 分頁

#### Bootstrap 4

現在分頁器產生的分頁連結預設會是 Bootstrap 4。要讓分頁器產生 Bootstrap 3 連結，可以從 `AppServiceProvider` 的 `boot` 方法呼叫 `Paginator::useBootstrapThree` 方法：

    <?php

    namespace App\Providers;

    use Illuminate\Pagination\Paginator;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 啟動任何應用程式服務。
         *
         * @return void
         */
        public function boot()
        {
            Paginator::useBootstrapThree();
        }
    }

### Resources

#### `original` 屬性

[resource 回應](/docs/5.6/eloquent-resources)的 `original` 屬性現在設為原生模型，而不是 JSON 字串與陣列。這可以在測試過程中更容易的檢查回應的模型。

### 路由

#### 回傳新建立的模型時

直接從路由回傳新建立的 Eloquent 模型會自動回應 `201` 的狀態，而不是 `200`。如果你的應用程式的任何測試有預期回應 `200`，則應該更新這些測試預期為 `201`。

### 可信任的代理

因為 Symfony HttpFoundation 的可信任代理功能的底層有更動，所以必須修改你的應用程式的中介層。

`$headers` 屬性在過去是一組陣列，現在則是一個 bit 屬性，可以接受不同的值。例如，要信任所有轉發的標頭，你可以把 `$headers` 屬性更新為下列的值：

    use Illuminate\Http\Request;

    /**
     * 被用於檢查代理的標頭。
     *
     * @var int
     */
    protected $headers = Request::HEADER_X_FORWARDED_ALL;

更多關於 `$headers` 值的資訊，請詳閱[信任代理](/docs/5.6/requests#configuring-trusted-proxies)上的所有文件。

### 驗證

#### `ValidatesWhenResolved` 介面

為了避免與 `$request->validate()` 方法衝突，已將 `ValidatesWhenResolved` interface／trait 重新命名為 `validateResolved`。

### 其他

我們也鼓勵你查看 `laravel/laravel` [GitHub repository](https://github.com/laravel/laravel) GitHub 儲存庫中的任何異動。儘管許多更改並不是必要的，但你可能希望保持這些文件與你的應用程序同步。其中一些更改將在本升級指南中介紹，但其他更改（例如更改設定檔案或註釋）將不會被介紹。你可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/5.5...5.6)來輕易的檢查更動的內容，並選擇哪些更新對你比較重要。
