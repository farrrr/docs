# 升級指南

- [從 6.x 升級到 7.0](#upgrade-7.0)

<a name="high-impact-changes"></a>
## 高度影響的變更

<div class="content-list" markdown="1">
- [Authentication Scaffolding](#authentication-scaffolding)
- [日期序列化](#date-serialization)
- [Symfony 5 相關升級](#symfony-5-related-upgrades)
</div>

<a name="medium-impact-changes"></a>
## 中度影響的變更

<div class="content-list" markdown="1">
- [Blade 元件與「Blade X」](#blade-components-and-blade-x)
- [支援 CORS](#cors-support)
- [工廠類型](#factory-types)
- [Markdown 信件模板升級](#markdown-mail-template-updates)
- [`Blade::component` 方法](#the-blade-component-method)
- [`assertSee` 斷言](#assert-see)
- [`different` 驗證規則](#the-different-rule)
- [唯一路由名字]](#unique-route-names)
</div>

<a name="upgrade-7.0"></a>
## 從 6.x 升級到 7.0

#### 預估升級時間：15 分鐘

> {note} 我們嘗試記錄每個重大變更。由於有些重大的變更是在框架最隱密的地方，但實際上只有一小部分的變更會影響你的應用程式。

### 需要 Symfony 5

**影響程度：高**

Laravel 7 已經升級底層的 Symfony 系列元件至 5.x，也作為當前最新的最低相容性版本。

### 需要 PHP 7.2.5

**影響程度：低**

PHP 版本最低需求為 7.2.5。

<a name="updating-dependencies"></a>
### 升級依賴項目

在 `composer.json` 檔升級你的 `laravel/framework` 依賴項目到 `^7.0`。此外，升級你的 `nunomaduro/collision` 依賴項目到 `^4.1`、 `phpunit/phpunit` 升級至 `^8.5`、`laravel/tinker` 升級至 `^2.0`、還有 `facade/ignition` 升級至 `^2.0`。

下列的官方套件已有新的版本來支援 Laravel 7。如果有安裝其中的套件，請在升級之前詳讀個別的升級指南：

- [Browser Kit Testing v6.0](https://github.com/laravel/browser-kit-testing/blob/master/UPGRADE.md)
- [Envoy v2.0](https://github.com/laravel/envoy/blob/master/UPGRADE.md)
- [Horizon v4.0](https://github.com/laravel/horizon/blob/master/UPGRADE.md)
- [Nova v3.0](https://nova.laravel.com/releases)
- [Scout v8.0](https://github.com/laravel/scout/blob/master/UPGRADE.md)
- [Telescope v3.0](https://github.com/laravel/telescope/releases)
- [Tinker v2.0](https://github.com/laravel/tinker/blob/2.x/CHANGELOG.md)
- UI v2.0 (不需要更改))

最後，請檢查你的應用程式所使用的第三方套件的版本是否能支援 Laravel 7。

<a name="symfony-5-related-upgrades"></a>
### Symfony 5 的相關升級

**影響程度：高**

Laravel 7 採用 Symfony 5.x 的元件系列。所以要適應這次的升級，還需要做一些小變更。

首先，`App\Exceptions\Handler` 類別的 `report`、`render`、`shouldReport` 和 `renderForConsole` 方法只接受 `Throwable` interface 的實例，而不是 `Exception` 實例：

    use Throwable;

    public function report(Throwable $exception);
    public function shouldReport(Throwable $exception);
    public function render($request, Throwable $exception);
    public function renderForConsole($output, Throwable $exception);

接著，請更新你的 `session` 設定檔的 `secure` 選項的預設值為 `null`：

    'secure' => env('SESSION_SECURE_COOKIE', null),

Symfony Console 是 Artisan 底層運行的主要元件，並預期所有指令都回傳一個整數。因此，你必須確保任何指令的回傳值都是整數。

    public function handle()
    {
        // 升級前...
        return true;

        // 升級後...
        return 0;
    }

### 認證

<a name="authentication-scaffolding"></a>
#### 起手式

**影響程度：高**

所有認證相關的前端套件都已經搬到 `laravel/ui` 儲存庫了。如果你有使用到 Laravel 的認證相關的前端套件，就需要在所有的專案中安裝該套件的 `^2.0` 版本。若你曾把這個套件設置在 `composer.json` 檔案的 `require-dev` 之中，請把它移至 `require` 的區塊：

    composer require laravel/ui "^2.0"

#### `TokenRepositoryInterface`

**影響程度：低**

`recentlyCreatedToken` 方法已經加入到 `Illuminate\Auth\Passwords\TokenRepositoryInterface` 介面。如果你正在撰寫這個介面的特定實作，你應該將該方法加入到你的實作中。

### Blade

<a name="the-blade-component-method"></a>
#### `component` 方法

**影響程度：中**

`Blade::component` 方法已經重新命名為 `Blade::aliasComponent`。請更新有用到這個方法的地方。

<a name="blade-components-and-blade-x"></a>
#### Blade 元件與「Blade X」

**影響程度：中**

Laravel 7 引入了官方支援的 Blade「標籤元件」。如果你想要停用 Blade 的內建標籤元件功能，可以從 `AppServiceProvider` 的 `boot` 方法上呼叫 `withoutComponentTags` 方法：

    use Illuminate\Support\Facades\Blade;

    Blade::withoutComponentTags();

### Eloquent

#### `addHidden` / `addVisible` 方法

**影響程度：低**

不在文件上的 `addHidden` 和 `addVisible` 方法已經正式移除了。並以 `makeHidden` 和 `makeVisible` 方法取而代之。

#### `booting` / `booted` 方法

**影響程度：低**

Eloquent 加入了 `booting` 和 `booted` 方法來更方便的定義模型在「boot」階段的任何邏輯。如果你有模型已經定義了這些方法，你需要將它們重新命名來避免與新加入的方法產生衝突。

<a name="date-serialization"></a>
#### 日期序列化

**影響程度：高**

在 Eloquent 模型上使用 `toArray` 或 `toJson` 方法的時候，Laravel 7 將採用新的日期序列化格式。框架現在使用 Carbon 的 `toJSON` 方法來格式化日期，這個方法可以產生包含時區資訊與小數秒的國際標準 ISO-8601 日期格式。此外，這個改變可以提供更好的支援與整合客戶端的解析日期的函式庫。

在過去，日期會被序列化成這個格式：`2019-12-02 20:01:00`。現在新的序列化後的日期格式會像是：`2019-12-02T20:01:00.283041Z`。請注意，國際標準 ISO-8601 始終以 UTC 表示。

如果你想要維持之前的做法，你可以覆寫模型的 `serializeDate` 方法：

    use DateTimeInterface;

    /**
     * Prepare a date for array / JSON serialization.
     *
     * @param  \DateTimeInterface  $date
     * @return string
     */
    protected function serializeDate(DateTimeInterface $date)
    {
        return $date->format('Y-m-d H:i:s');
    }

> {tip} 這個改變只會影響模型和模型集合被序列化成陣列與 JSON。這個改變並不影響如何在資料庫中儲存日期。

<a name="factory-types"></a>
#### 工廠類型

**影響程度：中**

Laravel 7 移除了「工廠類型」的功能。這個功能自 2016 年十月以來未被記錄在文件上。如果你還有在使用這個功能，你必須更換成[工廠狀態](/docs/{{version}}/database-testing#factory-states)，來提供更多的可塑性。

#### `getOriginal` 方法

**影響程度：低**

`$model->getOriginal()` 方法現在將不再優先於模型上定義的任何修改器與型別轉換器。在過去，這個方法會回傳轉換之前的原始屬性。如果你想要繼續取得原始未轉換的型別的值，你可以使用 `getRawOriginal` 方法來取代。

#### 路由綁定

**影響程度：低**

`Illuminate\Contracts\Routing\UrlRoutable` 介面的 `resolveRouteBinding` 方法接受 `$field` 參數。如果你是手動實作這個介面，則必須更新它。

還有，`Illuminate\Database\Eloquent\Model` 類別的 `resolveRouteBinding` 方法現在還接受 `$field` 的參數。如果你想要覆寫這個方法，就必須加入這個參數。

最後，`Illuminate\Http\Resources\DelegatesToResources` trait 的 `resolveRouteBinding` 方法現在也接受 `$field` 參數。如果你想要覆寫這個方法，就必須加入這個參數。

### HTTP

#### PSR-7 相容性

**影響程度：低**

不推薦使用 Zend Diactoros 函式庫來產生 PSR-7 的回應。如果正好使用這個套件來處理 PSR-7 的相容性，請改安裝 `nyholm/psr7` Composer 套件。還有，請安裝 `symfony/psr-http-message-bridge` 的 `^2.0` 版本的 Composer 套件。

### Mail

#### 設定檔的變更

**影響程度：非必要**

為了支援多個信件功能，在 Laravel 7.x 中的預設 `mail` 設定檔已改用 `mailers` 的陣列。然而，為了維持 6.x 的相容性，仍會繼續支援 Laravel 6.x 格式的設定檔。因此，升級到 Laravel 7.x 時不需要做任何的更改。但是，你可能想要[檢查新的 `mail` 設定檔](https://github.com/laravel/laravel/blob/develop/config/mail.php)架構並更新設定檔。

<a name="markdown-mail-template-updates"></a>
#### Markdown 信件模板更新

**影響程度：中**

預設的 Markdown 信件模板已汰換成更專業與吸引人的設計。此外，尚未記錄在正式文件的 `promotion` Markdown 信件元件已正式被移除了。

由於縮排在 Markdown 具有特別的意義，因此 Markdown 信件模板需要不縮排的 HTML。如果你之前已經發佈了 Laravel 的預設信件模板，則需要重新發佈信件模板或手動取消縮排：

    php artisan vendor:publish --tag=laravel-mail --force

#### Swift Mailer 綁定

**影響程度：低**

Laravel 7.x 不在提供 `swift.mailer` 和 `swift.transport` 容器綁定了。你現在可以透過 `mailer` 來存取綁定的物件：

    $swiftMailer = app('mailer')->getSwiftMailer();

    $swiftTransport = $swiftMailer->getTransport();

### Resource

#### `Illuminate\Http\Resources\Json\Resource` 類別

**影響程度：低**

現在已刪除 `Illuminate\Http\Resources\Json\Resource` 這個不推薦的類別。你的 Resource 應該改使用`Illuminate\Http\Resources\Json\JsonResource` 類別來繼承。

### 路由

#### 路由器的 `getRoutes` 方法

**影響程度：低**

路由器的 `getRoutes` 方法現在會回傳 `Illuminate\Routing\RouteCollectionInterface` 實例來取代 `Illuminate\Routing\RouteCollection`。

<a name="unique-route-names"></a>
#### 唯一的路由名字 Route Names

**影響程度：中**

儘管從未寫入正式文件中，但在過去的 Laravel 版本可以讓你為兩個不同的路由定義一樣的名字。在 Laravel 7 中，是無法再讓你這樣做，你必須為你的路由取一個不重複的名字。名字重複的路由會導致框架的許多地方產生非預期的行為。

<a name="cors-support"></a>
#### 支援 CORS

**影響程度：中**

現在預設支援已經整合好的跨來源資料共用（CORS）。如果你現在正在使用任何第三方的 CORS 函式庫，則建議你使用[新的 `cors` 設定檔](https://github.com/laravel/laravel/blob/master/config/cors.php).

接著，為應用程式的依賴項目安裝底層 CORS 函式庫：

    composer require fruitcake/laravel-cors

最後，加入 `\Fruitcake\Cors\HandleCors::class` 中介層到你的 `App\Http\Kernel` 全域中介層清單中。

### Session

#### 由 `array` 驅動的 Session

**影響程度：低**

由 `array` 驅動的 Session 資料現在會保留在當前的請求。在過去，即使還在請求階段，也無法取得儲存在 `array` Session 的資料。

### 測試

<a name="assert-see"></a>
#### `assertSee` 斷言

**影響程度：中**

在 `TestResponse` 類別上的 `assertSee`、`assertDontSee`、`assertSeeText`、`assertDontSeeText`、`assertSeeInOrder` 和 `assertSeeTextInOrder` 的斷言現在會自動跳脫值。如果你是手動跳脫任何傳入的值到這些斷言，則不用繼續這樣做了。如果你需要斷言未跳脫的值，可以在這些方法的第二個參數傳入 `false`。

<a name="test-response"></a>
#### `TestResponse` 類別

**影響程度：低**

`Illuminate\Foundation\Testing\TestResponse` 類別已重新命名為 `Illuminate\Testing\TestResponse`。如果你有使用過這個類別，請確保有更新這個命名空間。

<a name="assert-class"></a>
#### `Assert` 類別

**影響程度：低**

The `Illuminate\Foundation\Testing\Assert` class has been renamed to `Illuminate\Testing\Assert`. If you're using this class, make sure to update the namespace.

### 驗證

<a name="the-different-rule"></a>
#### `different` 規則

**影響程度：中**

現在，`different` 規則會因為請求中缺少指定的參數而失敗。

<a name="miscellaneous"></a>
### 其他

我們也鼓勵你查看 `laravel/laravel` [GitHub repository](https://github.com/laravel/laravel) GitHub 儲存庫中的任何異動。儘管許多更改並不是必要的，但你可能希望保持這些文件與你的應用程序同步。其中一些更改將在本升級指南中介紹，但其他更改（例如更改設定檔案或註釋）將不會被介紹。你可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/6.x...master)來輕易的檢查更動的內容，並選擇哪些更新對你比較重要。
