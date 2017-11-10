# Mocking

- [介紹](#introduction)
- [假 Bus](#bus-fake)
- [假事件](#event-fake)
- [假郵件](#mail-fake)
- [假通知](#notification-fake)
- [假隊列](#queue-fake)
- [假儲存](#storage-fake)
- [Facades](#mocking-facades)

<a name="introduction"></a>
## 介紹

測試 Laravel 應用程式時，你可能希望「模擬」應用程式的某些功能，而不會在特定的測試中實際執行它們。例如，測試控制器的觸發事件，你可能希望模擬物事件監聽器，而不是真的在測試中執行它們。這可以讓你只測試控制器的 HTTP 回應，而不用擔憂事件監聽器的執行，因為事件監聽器能在自己的測試案例中被測試。

Laravel 為模擬事件、任務和 facade 提供可立即使用的輔助函式。這些輔助函式主要提供方便的 Mockery 層，所以你不用手動寫些複雜的 Mockery 方法呼叫。當然，你可以自由使用 [Mockery](http://docs.mockery.io/en/latest/) 或 PHPUnit 來建立自己的 Mockery。

<a name="bus-fake"></a>
## 假 Bus

你可以使用 `Bus` facade 的 `fake` 方法作為 Mockery 的其中一個選擇，並用來預防任務真的被觸發。使用 `fake` 時，會等到測試的程式碼被執行後進行斷言：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use App\Jobs\ShipOrder;
    use Illuminate\Support\Facades\Bus;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class ExampleTest extends TestCase
    {
        public function testOrderShipping()
        {
            Bus::fake();

            // 執行訂單出貨...

            Bus::assertDispatched(ShipOrder::class, function ($job) use ($order) {
                return $job->order->id === $order->id;
            });

            // 斷言任務是否沒被觸發...
            Bus::assertNotDispatched(AnotherJob::class);
        }
    }

<a name="event-fake"></a>
## 假事件

你可以使用 `Event` facade 的 `fake` 方法作為 Mockery 的其中一個選擇，並用來防止所有事件監聽器被觸發。你可以斷言事件有沒有被觸發，甚至是檢查它們接收的資料。使用 `fake` 時，會等到測試的程式碼被執行後進行斷言：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use App\Events\OrderShipped;
    use App\Events\OrderFailedToShip;
    use Illuminate\Support\Facades\Event;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class ExampleTest extends TestCase
    {
        /**
         * Test order shipping.
         */
        public function testOrderShipping()
        {
            Event::fake();

            // 執行訂單出貨...

            Event::assertDispatched(OrderShipped::class, function ($e) use ($order) {
                return $e->order->id === $order->id;
            });

            Event::assertNotDispatched(OrderFailedToShip::class);
        }
    }

<a name="mail-fake"></a>
## 假郵件

你可以使用 `Mail` facade 的 `fake` 方法來預防郵件真的被寄出。你可以斷言 [mailables](/docs/{{version}}/mail) 有沒有寄給使用者，甚至是檢查它們接收的資料。使用 `fake` 時，會等到測試的程式碼被執行後進行斷言：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use App\Mail\OrderShipped;
    use Illuminate\Support\Facades\Mail;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class ExampleTest extends TestCase
    {
        public function testOrderShipping()
        {
            Mail::fake();

            // 執行訂單出貨...

            Mail::assertSent(OrderShipped::class, function ($mail) use ($order) {
                return $mail->order->id === $order->id;
            });

            // Assert a message was sent to the given users...
            Mail::assertSent(OrderShipped::class, function ($mail) use ($user) {
                return $mail->hasTo($user->email) &&
                       $mail->hasCc('...') &&
                       $mail->hasBcc('...');
            });

            // 斷言 mailable 是否寄了兩次...
            Mail::assertSent(OrderShipped::class, 2);

            // 斷言 mailable 是否沒寄出...
            Mail::assertNotSent(AnotherMailable::class);
        }
    }

如果你在背景執行隊列 mailables 的發送。你應該使用 `assertQueued` 方法，而不是 `assertSent`：

    Mail::assertQueued(...);
    Mail::assertNotQueued(...);

<a name="notification-fake"></a>
## 假通知

你可以使用 `Notification` facade 的 `fake` 方法來防止通知真的被送出。你可以斷言[通知](/docs/{{version}}/notifications) 是不是發送給使用者，甚至是檢查它們接收的資料。使用 `fake` 時，會等到測試的程式碼被執行後進行斷言：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use App\Notifications\OrderShipped;
    use Illuminate\Support\Facades\Notification;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class ExampleTest extends TestCase
    {
        public function testOrderShipping()
        {
            Notification::fake();

            // 執行訂單出貨...

            Notification::assertSentTo(
                $user,
                OrderShipped::class,
                function ($notification, $channels) use ($order) {
                    return $notification->order->id === $order->id;
                }
            );

            // 斷言通知是否發送給特定使用者...
            Notification::assertSentTo(
                [$user], OrderShipped::class
            );

            // 斷言通知是否沒送出...
            Notification::assertNotSentTo(
                [$user], AnotherNotification::class
            );
        }
    }

<a name="queue-fake"></a>
## 假隊列

你可以使用 `Queue` facade 的 `fake` 方法作為 Mockery 的其中一個選擇，並用來防止任務真的被隊列。你可以斷言任務是否被推送到隊列，甚至是檢查它們接收的資料。使用 `fake` 時，會等到測試的程式碼被執行後進行斷言：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use App\Jobs\ShipOrder;
    use Illuminate\Support\Facades\Queue;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class ExampleTest extends TestCase
    {
        public function testOrderShipping()
        {
            Queue::fake();

            // 執行訂單出貨...

            Queue::assertPushed(ShipOrder::class, function ($job) use ($order) {
                return $job->order->id === $order->id;
            });

            // 斷言任務是否被推送到特定隊列...
            Queue::assertPushedOn('queue-name', ShipOrder::class);

            // 斷言任務是否被推送兩次...
            Queue::assertPushed(ShipOrder::class, 2);

            // 斷言任務是否沒被退送...
            Queue::assertNotPushed(AnotherJob::class);
        }
    }

<a name="storage-fake"></a>
## 假儲存

`Storage` facade 的 `fake` 方法可以讓你輕易的產生假硬碟，並搭配 `UploadedFile` 類別的檔案產生工具，這可以大大簡化檔案上傳的測試。例如：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use Illuminate\Http\UploadedFile;
    use Illuminate\Support\Facades\Storage;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class ExampleTest extends TestCase
    {
        public function testAvatarUpload()
        {
            Storage::fake('avatars');

            $response = $this->json('POST', '/avatar', [
                'avatar' => UploadedFile::fake()->image('avatar.jpg')
            ]);

            // 斷言檔案使否被儲存...
            Storage::disk('avatars')->assertExists('avatar.jpg');

            // 斷言檔案是否存在...
            Storage::disk('avatars')->assertMissing('missing.jpg');
        }
    }

> {tip} 預設的 `fake` 方法會刪除在暫存目錄中的所有檔案。如果你希望保留這些檔案，你可以使用 `persistentFake` 方法來取代。

<a name="mocking-facades"></a>
## Facades

Facade 不像是傳統靜態方法的呼叫方式，[Facade](/docs/{{version}}/facades) 是可以被模擬的。這為傳統靜態方法提供一個很大的優點，並為你提供與使用依賴注入一樣的可測試性。你可能經常想模擬在控制器中呼叫 Laravel facade。例如，請參考以下控制器的操作：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\Cache;

    class UserController extends Controller
    {
        /**
         * 顯示應用程式所有使用者清單。
         *
         * @return Response
         */
        public function index()
        {
            $value = Cache::get('key');

            //
        }
    }

我們能使用 `shouldReceive` 來模擬呼叫 `Cache` facade，這會回傳 [Mockery](https://github.com/padraic/mockery) 模擬實例。因為 facades 實際上是被 Laravel [服務容器](/docs/{{version}}/container)所管理與解析，它們比典型的靜態類別更具可測試性。例如，讓我們模擬呼叫 `Cache` facade 的 `get` 方法：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use Illuminate\Support\Facades\Cache;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class UserControllerTest extends TestCase
    {
        public function testGetIndex()
        {
            Cache::shouldReceive('get')
                        ->once()
                        ->with('key')
                        ->andReturn('value');

            $response = $this->get('/users');

            // ...
        }
    }

> {note} 你不應該模擬 `Request` facade。而是傳入你想要的輸入到 HTTP 輔助函式方法，像是在執行測試時的 `get` 和 `post`。同樣地，不要模擬 `Config` facade，只要在你的測試中呼叫 `Config::set` 方法。
