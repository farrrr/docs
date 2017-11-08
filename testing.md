# 測試：入門

- [介紹](#introduction)
- [環境](#environment)
- [建立並執行測試](#creating-and-running-tests)

<a name="introduction"></a>
## 介紹

Laravel 有考慮到測試這件事。實際上，早已支援 PHPUnit 測試且能馬上使用，以及 `phpunit.xml` 準備好為你設定應用程式設定測試了。框架還附帶便捷的輔助函示方法，可以讓你更直觀的測試應用程式。

預設應用程式的 `tests` 目錄會有兩個子目錄：`Feature` 和 `Unit`。單元測試主要測試非常小且能獨立於你的程式碼部分。實際上，大多數的單元測試可能只專注於單一個方法。功能測試可以測試大部分的程式碼，包含幾個物件如何進行交互，甚至是完整的 HTTP 請求到 JSON 端。

`Feature` 和 `Unit` 測試目錄都提供了一個名為 `ExampleTest.php` 的檔案。安裝新 Laravel 應用程式後，在指令列上執行 `phpunit` 來開始你的測試。

<a name="environment"></a>
## 環境

透過 `phpunit` 執行測試時，因為在 `phpunit.xml` 檔案定義環境變數，Laravel 會自動將環境設定為 `testing`。Laravel 還會在測試時自動將 Session 和快取設定到 `array` 驅動，也就是說不會保留在測試時的 Session 或快取資料。

你是可以自由的根據需要來定義其他測試環境設定值。`testing` 環境變數可以被設定在 `phpunit.xml` 檔案中，但是要確保在執行測試前有執行 Artisan 指令的 `config:clear` 來清除設定檔的快取！

<a name="creating-and-running-tests"></a>
## 建立並執行測試

要建立一個新的測試案例，請使用 Artisan 指令的 `make:test`：

    // 在功能測試目錄建立一個測試...
    php artisan make:test UserTest

    // 在單元測試目錄建立一個測試...
    php artisan make:test UserTest --unit

測試一旦被產生，你可以像是在使用 PHPUnit 來定義測試方法。若要執行測試，只要在終端機執行 `phpunit` 指令：

    <?php

    namespace Tests\Unit;

    use Tests\TestCase;
    use Illuminate\Foundation\Testing\RefreshDatabase;

    class ExampleTest extends TestCase
    {
        /**
         * 基礎測試範例。
         *
         * @return void
         */
        public function testBasicTest()
        {
            $this->assertTrue(true);
        }
    }

> {note} 如果你在測試類別中定義自己的 `setUp` 方法，請別忘了呼叫 `parent::setUp()`。
