# 廣播系統

- [介紹](#introduction)
    - [設定](#configuration)
    - [驅動需求](#driver-prerequisites)
- [概念簡述](#concept-overview)
    - [示範一個例子](#using-example-application)
- [定義廣播事件](#defining-broadcast-events)
    - [廣播名稱](#broadcast-name)
    - [廣播資料](#broadcast-data)
    - [廣播隊列](#broadcast-queue)
    - [廣播條件](#broadcast-conditions)
- [頻道授權](#authorizing-channels)
    - [定義授權的路由](#defining-authorization-routes)
    - [定義授權的回呼函式](#defining-authorization-callbacks)
- [廣播事件](#broadcasting-events)
    - [只廣播給其他人](#only-to-others)
- [接受廣播](#receiving-broadcasts)
    - [安裝 Laravel Echo](#installing-laravel-echo)
    - [監聽事件](#listening-for-events)
    - [退出頻道](#leaving-a-channel)
    - [命名空間](#namespaces)
- [Presence 頻道](#presence-channels)
    - [認證給 Presence 頻道](#authorizing-presence-channels)
    - [加入到 Presence 頻道](#joining-presence-channels)
    - [廣播到 Presence 頻道](#broadcasting-to-presence-channels)
- [客戶端事件](#client-events)
- [通知系統](#notifications)

<a name="introduction"></a>
## 介紹

在許多現代網頁應用程式中，WebSocket 被用來實作 realtime 與即時更新的使用者介面。當伺服器在更新某些資料時，會透過 WebSocket 的連線去處理使用者的訊息。相較於不斷地重新查詢來取得後端資料的方式，這提供了更強大、更有效率的方式。

為了協助你建構 WebSocket 連線，Laravel 把「廣播[事件](/docs/{{version}}/events)」弄的很輕鬆。 Laravel 允許你在廣播事件的時候共用後端與前端（JavaScript）的事件名稱。

> {tip} 再深入了解廣播之前，請你先閱讀所有關於 Laravel [事件與監聽器](/docs/{{version}}/events) 的文件。

<a name="configuration"></a>
### 設定

所有關於事件廣播設定都存放在 `config/broadcasting.php` 這個設定檔。Laravel 支援幾個廣播用的驅動：[Pusher](https://pusher.com)、[Redis](/docs/{{version}}/redis) 和一個用來本地開發與除錯的 `log`。此外，你可以使用 `null` 來禁用所有的廣播器。在 `config/broadcasting.php` 中，每個驅動的配置都有附上設定的範例供你參考。

#### 廣播的服務提供者

在廣播任何事件前，首先你需要註冊 `App\Providers\BroadcastServiceProvider`。在全新的 Laravel 應用程式中，你只需要在 `config/app.php` 設定檔中找到 `providers` 陣列，並取消對它們的註解。這個提供者可以讓你註冊廣播授權路由和回呼函式。

#### CSRF Token

[Laravel Echo](#installing-laravel-echo) 會需要存取當前 session 的 CSRF token 。你應當確認你的 `head` HTML 標籤裡是否有放入 `meta` 標籤來設定 CSRF token：

    <meta name="csrf-token" content="{{ csrf_token() }}">

<a name="driver-prerequisites"></a>
### 驅動需求

#### Pusher

如果你透過 [Pusher](https://pusher.com) 來廣播事件，你應該使用 Composer 安裝 Pusher PHP SDK：

    composer require pusher/pusher-php-server "~3.0"

接下來，你需要設定你的 Pusher 憑證到 `config/broadcasting.php` 這個設定檔。這個文件已寫好 Pusher 設定範例，你只需要去修改預設的 Pusher 金鑰 、密碼和應用程式 ID 。 `config/broadcasting.php` 裡面的 `Pusher` 設定允許你使用 `options` 來加入 Puhser 支援的額外功能選項，像是下面寫的：

    'options' => [
        'cluster' => 'eu',
        'encrypted' => true
    ],

在同時使用Pushser 和 [Laravel Echo](#installing-laravel-echo) 的時候，你應該在 `resources/assets/js/bootstrap.js` 檔案中實體化 Echo 實例的時候，將指定 Pusher 作為所需的廣播器：

    import Echo from "laravel-echo"

    window.Pusher = require('pusher-js');

    window.Echo = new Echo({
        broadcaster: 'pusher',
        key: 'your-pusher-key'
    });

#### Redis

如果你使用 Redis 作為你的廣播器，你應該安裝 Predis 套件：

    composer require predis/predis

Redis 廣播器會使用 Redis 發布/訂閱 功能來廣播訊息。然而，你還是需要搭配 WebSocket 來接受來自 Redis 的訊息，並廣播它們到 WebSocket 的頻道

當 Redis 廣播器發布一個事件時，該事件會被發佈到指定的頻道上，裝載的資料會是 JSON 格式，並包含了事件名稱、 `data` 和使用者產生的事件 Socket ID（如果需要的話）：

#### Socket.IO

如果你搭配 Redis 和 Socket.IO ，你會需要載入 Socket.IO 的 JavaScript 程式庫在你網頁的 `head` HTML 標籤裡。當 Socket.IO 伺服器啟動時，它會自動公開客戶端的 JavaScript 程式庫的 URL。舉例來說，如果你執行的 Socket.IO 和網頁在同一個網域，你可以像下面這樣來讓前端存取你的 Socket.IO 的 JavaScript 程式庫：

    <script src="//{{ Request::getHost() }}:6001/socket.io/socket.io.js"></script>

接著，你會需要指定 `socket.io` 和 `host` 連線來實例化 Echo。

    import Echo from "laravel-echo"

    window.Echo = new Echo({
        broadcaster: 'socket.io',
        host: window.location.hostname + ':6001'
    });

最後，你會需要可以相容於 Socket.IO 的伺服器。Laravel 不會載入 Socket.IO 伺服器來實作。然而，有個由社群維護與提供的 Socket.IO伺服器—— [tlaverdure/laravel-echo-server](https://github.com/tlaverdure/laravel-echo-server)。

#### Queue 必備條件

在廣播事件之前，你也許會需要設定和執行[隊列監聽器](/docs/{{version}}/queues)。全部的事件廣播都會通過隊列器進行，使你的程式回應時間不會受到太嚴重的影響。


<a name="concept-overview"></a>
## 觀念簡述

Laravel 的事件廣播允許你使用基於驅動的 WebSockets 將後端的 Laravel 事件廣播給前端 JavaScript 應用程式。目前，Laravel 內建了 [Pusher](https://pusher.com) 和 Redis 的驅動。在前端使用 [Laravel Echo](#installing-laravel-echo) Javascript 的套件，可以更簡單的處理事件

事件通過「頻道」來廣播，頻道可以被指定為公開或私人的。你的應用程式的任何訪客都可以訂閱公共頻道，且不需要在認證身份或檢查授權。然而，如果為了訂閱一個私人頻道，使用者就必須認證身份與通過授權才可以監聽該頻道。

<a name="using-example-application"></a>
### 示範一個例子

在深入廣播事件的每個元件前，讓我們用電子商務作為較完整的例子。在這邊，我們不會討論如何配置 [Pusher](https://pusher.com) 或 [Laravel Echo](#installing-laravel-echo)，因為這些內容會在技術文件的其他部分做詳細討論。

在我們的應用程式中，假設我們有一個頁面，提供給使用者查看訂單的運送狀況。我們還假定當應用程式處理運送狀態更新的時候，會發出 `ShippingStatusUpdated` 的事件：

    event(new ShippingStatusUpdated($update));

####  `ShouldBroadcast` 介面

當使用者正在查看其中一個訂單時，我們並不想要讓他們透過「重新整理」這個頁面的方式來檢視狀態更新與否。換言之，我們想要在他們建立訂單時透過廣播去更新狀態。所以，我們需要在 `ShouldBroadcast` 介面標記 `ShippingStatusUpdated` 事件。這會讓 Laravel 在事件觸發時，廣播這個事件：

    <?php

    namespace App\Events;

    use Illuminate\Broadcasting\Channel;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Broadcasting\PrivateChannel;
    use Illuminate\Broadcasting\PresenceChannel;
    use Illuminate\Broadcasting\InteractsWithSockets;
    use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

    class ShippingStatusUpdated implements ShouldBroadcast
    {
        /**
         * Information about the shipping status update.
         *
         * @var string
         */
        public $update;
    }

`ShouldBroadcast` 介面需要我們的事件去定義 `broadcastOn` 方法。這個方法負責回傳事件應該廣播到哪個頻道。已經在每個新建的事件類別中定義了這一個準備被填入細節的方法。我們只想要創建訂單的使用者能看到狀態的更新，所以我們會在與訂單相關的私人頻道上廣播這則事件：

    /**
     * Get the channels the event should broadcast on.
     *
     * @return array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('order.'.$this->update->order_id);
    }

#### 認證頻道

請記得，使用者最好需要授權機制去監聽私人頻道。，我們可以在 `routes/channels.php` 定義我們頻道的授權規則。在這個範例中，我們需要驗證任何在 `order.1` 的私人頻道上嘗試監聽的使用者是否是該訂單的持有人：

    Broadcast::channel('order.{orderId}', function ($user, $orderId) {
        return $user->id === Order::findOrNew($orderId)->user_id;
    });

`channel` 方法接受兩個參數：一個是頻道名稱，另一個是用來回傳 `true` 或 `false` 的回呼，這個回呼用來表示使用者是否被授權可以在頻道上監聽。

所有的授權回呼都會將當前認證的使用者作為第一個參數和任何額外的 wildcard 參數作為後續的參數。在本例中，我們使用 `{orderId}` 來表示頻道名稱的「 ID 」。

#### 監聽廣播事件

接著，就只剩下在 JavaScript 中監聽事件了。我們在這能使用 Laravel Echo。首先，我們會使用 `private` 方法去訂閱私人頻道。然後，我們可以使用 `listen` 方法來監聽 `ShippingStatusUpdated` 事件。在預設上，所有事件的公共屬性會被載入廣播事件中：

    Echo.private(`order.${orderId}`)
        .listen('ShippingStatusUpdated', (e) => {
            console.log(e.update);
        });

<a name="defining-broadcast-events"></a>
## 定義廣播事件

要告訴 Laravel 廣播給定的事件，只需要在事件類別上繼承 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 這個介面。該介面已經導入到所有由 Laravel 產生的事件類別中，所以你可以簡單地將它添加到任何事件中。

`ShouldBroadcast` 介面需要你實作一個方法：`broadcastOn`。`broadcastOn` 方法會回傳一個頻道或一組頻道的陣列，事件會被廣播道這些頻道上。頻道會是 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的這些實例。頻道會有三種實例：`Channel`、`PrivateChannel` 和 `PresenceChannel`。`Channel` 實例代表任何使用者都可以訂閱的公共頻道，而 `PrivateChannels` 和 `PresenceChannels` 則代表需要[頻道授權](#authorizing-channels)的私人頻道：

    <?php

    namespace App\Events;

    use Illuminate\Broadcasting\Channel;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Broadcasting\PrivateChannel;
    use Illuminate\Broadcasting\PresenceChannel;
    use Illuminate\Broadcasting\InteractsWithSockets;
    use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

    class ServerCreated implements ShouldBroadcast
    {
        use SerializesModels;

        public $user;

        /**
         * Create a new event instance.
         *
         * @return void
         */
        public function __construct(User $user)
        {
            $this->user = $user;
        }

        /**
         * Get the channels the event should broadcast on.
         *
         * @return Channel|array
         */
        public function broadcastOn()
        {
            return new PrivateChannel('user.'.$this->user->id);
        }
    }

然後，你只需要和平常一樣[觸發事件](/docs/{{version}}/events)。一旦事件被觸發，[隊列任務](/docs/{{version}}/queues)會通過你指定的廣播驅動自動廣播事件。

<a name="broadcast-name"></a>
### 廣播名稱

預設上，Laravel 會使用事件類別名稱去廣播事件。然而，你可以在事件上定義 `broadcastAs` 方法來自訂廣播名稱：

    /**
     * The event's broadcast name.
     *
     * @return string
     */
    public function broadcastAs()
    {
        return 'server.created';
    }

如果你使用 `broadcastAs` 方法自訂廣播名稱使用，你應當確保註冊你的監聽器字元前綴加上 `.`，這會告知 Laravel Echo 不要把應用程式的命名空間添加到事件中：

    .listen('.server.created', function (e) {
        ....
    });


<a name="broadcast-data"></a>
### 廣播資料

當廣播事件時，該事件所有的 `Public` 屬性會自動序列化並作為事件的資料來廣播，從而允許你存取來自 JavaScript 的任何公共資料。所以舉例來說，如果你的事件有一個 public `$user` 屬性，它包含了一個 Eloquent 模型，那麼事件的廣播資料會是：

    {
        "user": {
            "id": 1,
            "name": "Patrick Stewart"
            ...
        }
    }

然而，如果你希望對廣播資料有更細微的控制，你可以加入 `broadcastWith` 方法到你的事件中。這個方法會回傳一組陣列資料，並如你所希望的廣播事件資料：

    /**
     * Get the data to broadcast.
     *
     * @return array
     */
    public function broadcastWith()
    {
        return ['id' => $this->user->id];
    }

<a name="broadcast-queue"></a>
### 廣播隊列

預設上，每個廣播事件都放在預設隊列上，這些預設的指定隊列的連線在 `queue.php` 設定檔。你可以在你的事件類別上自訂定義 `broadcastQueue` 屬性來使用隊列。這個屬性會指定你希望在廣播的時候要用的隊列名稱：

    /**
     * The name of the queue on which to place the event.
     *
     * @var string
     */
    public $broadcastQueue = 'your-queue-name';

如果你想要使用 `sync` 隊列而不是使用預設的隊列驅動來廣播你的事件，你能實作 `ShouldBroadcastNow` 介面而不是 `ShouldBroadcast` 介面:

    <?php

    use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;

    class ShippingStatusUpdated implements ShouldBroadcastNow
    {
        //
    }
<a name="broadcast-conditions"></a>
### 廣播條件

有些時候，你只想在給定條件為 `true`時，才廣播該事件。你可以在你的事件類別上定義 `broadcastWhen` 方法，並添加一些你想要的條件陳述式：

    /**
     * Determine if this event should broadcast.
     *
     * @return bool
     */
    public function broadcastWhen()
    {
        return $this->value > 100;
    }

<a name="authorizing-channels"></a>
## 授權給頻道

私人頻道只允許你授權過的使用者才可以監聽頻道。這個過程是使用者發送一個 HTTP 請求到指定的 Laravel 的頻道名稱，並判斷該使用者是否可以監聽該頻道。當你在使用 [Laravel Echo](#installing-laravel-echo) 的時候，將會自行發送 HTTP 請求到想要訂閱的私人頻道。然而，你需要定義路由來回應這些請求。

<a name="defining-authorization-routes"></a>
### 定義授權的路由

所幸，Laravel 可以輕易地定義路由來回應頻道授權的請求。在 Laravel 載入的 `BroadcastServiceProvider`  這個提供器，你會看到呼叫 Broadcast::routes` 方法。該方法會註冊 `/broadcasting/auth` 路由來處理授權請求：

    Broadcast::routes();

`Broadcast::routes` 方法會自行將它的路由放置 `web` 中介層群組中。然而，如果你想要自定一些屬性，你可以傳遞一組路由屬性的陣列到這個方法：

    Broadcast::routes($attributes);

<a name="defining-authorization-callbacks"></a>
### 定義授權的回呼函式

接下來，我們需要定義實際執行頻道授權的邏輯。這是在 Laravel 內建的 `routes/channels.php` 檔案中完成。在這個檔案，你可以使用 `Broadcast::channel` 方法註冊頻道授權的回呼：

    Broadcast::channel('order.{orderId}', function ($user, $orderId) {
        return $user->id === Order::findOrNew($orderId)->user_id;
    });

`channel` 方法接受兩個參數：其一是頻道名稱，另一個是回傳 `true` 或 `false`，用來判斷使用者是否有授權監聽頻道的回呼。

所有的授權回呼都會將當前已認證使用者作為第一個參數，和任何其他萬元字元作為後續參數。在這個範例中，我們使用  `{orderId}` 來表示頻道名稱的「 ID 」。

#### 授權回呼模型綁定Authorization Callback Model Binding

就像 HTTP 路由，頻道路由也可以使用隱式和顯示[路由模型綁定](/docs/{{version}}/routing#route-model-binding)。舉例來說，相對於接收字串或數字的 Order ID，你可以請求實際的 `Order` 模型實例：

    use App\Order;

    Broadcast::channel('order.{order}', function ($user, Order $order) {
        return $user->id === $order->user_id;
    });

<a name="broadcasting-events"></a>
## 廣播事件

如果你在一個事件類別上實作 `ShouldBroadcast` 介面，那麼你只需要使用 `event` 函式來觸發事件。事件發送器會注意到該事件有實作 `ShouldBroadcast` 介面，並將事件加入到隊列等待廣播：

    event(new ShippingStatusUpdated($update));

<a name="only-to-others"></a>
### 只廣播給其他人

當建構一個正在事件廣播的應用程式時，你可以用 `broadcast` 函式來替換 `event` 函式，像 `event` 函式一樣， `broadcast` 函式把事件分派到你的後端監聽器上：

    broadcast(new ShippingStatusUpdated($update));

然而， `broadcast` 函式還有個 `toOthers` 方法可以讓你把當前使用者從廣播接收名單中排除：

    broadcast(new ShippingStatusUpdated($update))->toOthers();

為了更好理解何種情況會運用到 `toOthers` 方法，讓我們設想一個任務清單的應用程式，使用者可以輸入任務名稱來建立新的任務。要建立一個任務，首先你的應用程式會送出一個建立新任務的請求到 `/task` 路由，接著會觸發事件並廣播給使用者送出請求的結果，回傳的內容會 JSON 格式表示。當你的 JavaScript 應用程式接收到路由回應的內容，它會直接將剛剛建立成功的新任務插入到任務列表中。像是底下的例子：

    axios.post('/task', task)
        .then((response) => {
            this.tasks.push(response.data);
        });

然而，請記得我們剛廣播了這個任務的新建立事件。如果你的 JavaScript 應用程式同時也在監聽同一件事，你會得到一組重複的內容在同一個任務清單上：一個來自路由的，另一個來自廣播。

這時你就可以使用 `toOthers` 方法來告訴廣播器不要再廣播給觸發事件的使用者來解決這個問題。

#### 設定

在你初始化 laravel-echo 實例的時候，socket ID 會被指定到該連線上。如果你是使用 [Vue](https://vuejs.org) 和 [Axios](https://github.com/mzabriskie/axios) 這個組合，socket ID 會自動添加 `X-Socket-ID` 標頭到每個送出的請求上。然後，當你呼叫 `toOthers` 方法時，Laravel 會從標頭中拿到 socket ID，並告訴廣播器不要廣播到同一個 socket ID 的連線上。

如果你不是使用 Vue 和 Axios，你需要自行設定 JavaScript 應用程式來送出 `X-Socket-ID` 標頭。你可以使用 `Echo.socketId` 方法來取得 socket ID：

    var socketId = Echo.socketId();

<a name="receiving-broadcasts"></a>
## 接收廣播

<a name="installing-laravel-echo"></a>
### 安裝 Laravel Echo

Laravel Echo 是一個 JavaScript 程式庫，它可以無縫地在 Laravel 上訂閱頻道與監聽事件廣播。你可以透過 NPM 套件管理器來安裝 Echo。在這個範例中，我們也會安裝 `pusher-js` 套件，因為我們會使用 Pusher 來廣播：

    npm install --save laravel-echo pusher-js

一旦安裝好 Echo，你可以準備在你的 JavaScript 應用程式中建立全新的 Echo 實例。好消息是 Laravel 以為你寫好並放在 `resources/assets/js/bootstrap.js` 檔案的最底下：

    import Echo from "laravel-echo"

    window.Echo = new Echo({
        broadcaster: 'pusher',
        key: 'your-pusher-key'
    });

在使用 `pusher` 來建立 Echo 實例時，你還可以指定 `cluster` 以及是否需要加密連線：

    window.Echo = new Echo({
        broadcaster: 'pusher',
        key: 'your-pusher-key',
        cluster: 'eu',
        encrypted: true
    });

<a name="listening-for-events"></a>
### 監聽事件

在你安裝並實例化 Echo 的剎那，你就可以準備監聽事件廣播囉。首先，使用 `channel` 方法來取得一個頻道實例，然後呼叫 `listen` 方法去監聽指定的事件：

    Echo.channel('orders')
        .listen('OrderShipped', (e) => {
            console.log(e.order.name);
        });

如果你想在私人頻道上監聽事件，可以使用 `private` 方法。你可以繼續串接用來呼叫的 `listen` 方法去監聽同個頻道上的多個事件：

    Echo.private('orders')
        .listen(...)
        .listen(...)
        .listen(...);

<a name="leaving-a-channel"></a>
### 離開頻道

想要離開頻道，你可以在你的 Echo 實例上呼叫 `leave` 方法：

    Echo.leave('orders');

<a name="namespaces"></a>
### 命名空間

你可能注意到上述範例中沒有為事件類別指定完整的命名空間。這是因為 Echo 會自動假設事件會在 `App\Events` 這個命名空間裡。話雖如此，你可以在實例化 Echo 的時候傳遞 `namespace` 設定選項來設定想要的命名空間：

    window.Echo = new Echo({
        broadcaster: 'pusher',
        key: 'your-pusher-key',
        namespace: 'App.Other.Namespace'
    });

或者，當你在使用 Echo 的時候，可以使用 `.` 來前綴事件類別。這會允許你總是指定完整的類別名稱：

    Echo.channel('orders')
        .listen('.Namespace.Event.Class', (e) => {
            //
        });

<a name="presence-channels"></a>
## Presence 頻道

Presence 頻道建構在私人頻道的安全性上，同時多了一個「知道誰被訂閱在頻道上」的額外功能。這使得它可以輕易地建構既強大且功能。像是當有個使用者也在看相同頁面時，會通知其他使用者。

<a name="authorizing-presence-channels"></a>
### 授權給 Presence 頻道

所有的 presence 頻道也算是私人頻道。因此，使用者必須[被授權才可存取他們](#authorizing-channels)。然而，在定義 presence 頻道授權回呼的時候，若有個使用者有被授權進入頻道，則回傳 `true`，反之，你應該回傳一組關於使用者資料的陣列。

授權回呼所回傳的資料會用在 JavaScript 應用程式中的 presence 頻道事件監聽器。如果使用者未被授權允許加入 presence 頻道，你應該會傳 `false` 或 `null` ：

    Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
        if ($user->canJoinRoom($roomId)) {
            return ['id' => $user->id, 'name' => $user->name];
        }
    });

<a name="joining-presence-channels"></a>
### 加入到 Presence 頻道

要加入 presence 頻道，你可以使用 Echo 的 `join` 方法。 `join` 方法會回傳一個 `PresenceChannel` 實作，並且你還可以繼續加上 `listen` 方法，這會允許你使用訂閱 `here`、`joining` 和 `leaving` 事件。

    Echo.join(`chat.${roomId}`)
        .here((users) => {
            //
        })
        .joining((user) => {
            console.log(user.name);
        })
        .leaving((user) => {
            console.log(user.name);
        });

如果頻道連線成功，`here` 回呼就會立即執行，然後會接收到一組包含當前訂閱頻道的所有使用者的資訊。在一個新使用者加入頻道的時候會執行 `joining` 方法，並且在使用者離開頻道的時候執行 `leaving` 方法。

<a name="broadcasting-to-presence-channels"></a>
### 廣播到 Presence 頻道

Presence 頻道就像私人或公共頻道一樣，可以接收事件。來舉個聊天室的範例，我們可能想要廣播 `NewMessage` 事件到聊天室的 presence 頻道。為此，我們將從事件的 `broadcastOn` 方法回傳一個 `PresenceChannel` 實例：

    /**
     * Get the channels the event should broadcast on.
     *
     * @return Channel|array
     */
    public function broadcastOn()
    {
        return new PresenceChannel('room.'.$this->message->room_id);
    }

像是公共或私人頻道上的事件， presence 頻道上的事件可以使用 `broadcast` 函式來廣播。與其他事件一樣，你可以使用 `toOthers` 方法來將使用者從廣播接收名單中排除。

    broadcast(new NewMessage($message));

    broadcast(new NewMessage($message))->toOthers();

你可以透過 `listen` 方法來監聽 join 事件：

    Echo.join(`chat.${roomId}`)
        .here(...)
        .joining(...)
        .leaving(...)
        .listen('NewMessage', (e) => {
            //
        });

<a name="client-events"></a>
## 客戶端事件

有些時候，你可能希望在沒碰到 Laravel 的情況下發送一個事件到其他已連線的客戶端。這對於「正在輸入中」的通知格外有用，像是你想要在視窗上提醒使用者另外一個使用者正在輸入訊息。要廣播客戶端事件，你可以使用 Echo 的 `whisper` 方法：

    Echo.channel('chat')
        .whisper('typing', {
            name: this.user.name
        });

若要監聽客戶端事件，你可以使用 `listenForWhisper` 方法：

    Echo.channel('chat')
        .listenForWhisper('typing', (e) => {
            console.log(e.name);
        });

<a name="notifications"></a>
## 通知

事件廣播搭配[通知系統](/docs/{{version}}/notifications)，可讓你的 JavaScript 應用程式在不需要重新整理頁面得情況下接收新的通知。
首先，請看一下有關使用[廣播通知頻道](/docs/{{version}}/notifications#broadcast-notifications)的技術文件。

如果你已設定通知系統到廣播頻道，你就可以使用 Echo 的 `notification` 方法來監聽廣播事件。但請記得，頻道名稱應該與接收通知的實例之類別名稱一致：

    Echo.private(`App.User.${userId}`)
        .notification((notification) => {
            console.log(notification.type);
        });

在這個範例中，凡是透過廣播頻道發送到 `App\User` 實例的所有通知都將會被回呼所接收。`App.User.{id}` 頻道的授權回呼已載入到預設的 `BroadcastServiceProvider`。
