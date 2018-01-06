# 隊列

- [簡介](#introduction)
    - [連接 Vs. 隊列](#connections-vs-queues)
    - [隊列驅動基本要求](#driver-prerequisites)
- [建立任務](#creating-jobs)
    - [產生任務類別](#generating-job-classes)
    - [類別結構](#class-structure)
- [執行任務](#dispatching-jobs)
    - [延遲執行](#delayed-dispatching)
    - [隊列任務鏈](#job-chaining)
    - [自訂隊列 & 連接](#customizing-the-queue-and-connection)
    - [指定最大任務嘗試次數 / 逾時](#max-job-attempts-and-timeout)
    - [限制執行比例](#rate-limiting)
    - [錯誤處理](#error-handling)
- [Running The Queue Worker](#running-the-queue-worker)
    - [Queue Priorities](#queue-priorities)
    - [Queue Workers & Deployment](#queue-workers-and-deployment)
    - [Job Expirations & Timeouts](#job-expirations-and-timeouts)
- [設定 Supervisor](#supervisor-configuration)
- [處理失敗的任務](#dealing-with-failed-jobs)
    - [Cleaning Up After Failed Jobs](#cleaning-up-after-failed-jobs)
    - [Failed Job Events](#failed-job-events)
    - [Retrying Failed Jobs](#retrying-failed-jobs)
- [任務事件](#job-events)

<a name="introduction"></a>
## 簡介

> {tip} Laravel 目前支援了 Horizon。為 Redis 運作的隊列提供一個美觀儀表板介面的設定系統。查看完整的 [Horizon 文件](/docs/{{version}}/horizon) 以取得更多的資訊。

Laravel 隊列為各式各樣的隊列後端服務提供了一個統一的 API，像是 Beanstalk、Amazon SQS、Redis 甚至是關聯式資料庫。隊列允許你抽離需要花較多時間處理的任務，例如經過一段時間後寄送一封 email。抽離這些較為耗時的任務能夠明顯地為你的網站應用程式加快處理每一個網頁請求。

隊列的設定檔位於 `config/queue.php`。在這個檔案內你可以馬上找到隊列連接的相關驅動設定，包含框架、資料庫、[Beanstalkd](https://kr.github.io/beanstalkd/)、[Amazon SQS](https://aws.amazon.com/sqs/)、[Redis](https://redis.io) 和以本機同步驅動執行任務的設定。使用 `null` 隊列驅動則可以很容易的丟棄所有隊列任務。

<a name="connections-vs-queues"></a>
### 連接 Vs. 隊列

在使用 Laravel 隊列之前，必須先明確了解「連接」與「隊列」之間的區別。在 `config/queue.php` 設定檔中有一個 `connections` 的設定選項，這個選項定義了連接隊列的後端服務，像是 Amazon SQS、Beanstalk 或是 Redis。每一個定義的「連接、，可以擁有多個不同的「隊列」，隊列內有許多的隊列任務，堆疊起來。

注意在 `queue` 設定檔中每個連接設定範例的 `queue` 屬性。將任何隊列任務送至該連接時，所有的隊列任務預設都會送到這個隊列內。換句話說，如果妳執行一個隊列任務時沒有指定任何的隊列，該隊列任務預設就會放置在設定檔裡 `queue` 屬性內預設定義的隊列內：

    // 這個隊列任務會被送至預設的隊列...
    Job::dispatch();

    // 這個隊列任務會被送至 "email" 隊列...
    Job::dispatch()->onQueue('emails');

一些應用程序可能無法滿足只使用單一個隊列，會需要推送任務至多個隊列內。針對應用程序需要設計哪些任務必須先執行或是切割任務時，推送任務至多個隊列的功能就特別好用。Laravel queue worker 允許你指定不同隊列執行任務的優先權。例如，若你推送任務至 `high` 隊列，你可以在執行 worker 時給予較高的處理順序：

    php artisan queue:work --queue=high,default

<a name="driver-prerequisites"></a>
### 隊列驅動基本要求

#### 資料庫

若要使用 `database` 作為隊列驅動，你必須先建置好一個資料表用來放置任務。你可以使用 Artisan 指令 `queue:table` 來產生這個資料表。當資料表遷移成功被建立後，你可以使用 `migrate` 指令進行資料庫的資料表遷移更新：

    php artisan queue:table

    php artisan migrate

#### Redis

若要使 `redis` 作為隊列驅動，你必須在 `config/database.php` 設定檔內設定你的 Redis 資料庫連接。

若你的 Redis 隊列連接使用 Redis 叢集，你的隊列名稱必須包含 [key hash tag](https://redis.io/topics/cluster-spec#keys-hash-tags)。這個選項是必須的，用來確保指定隊列的所有 Redis keys 被放置在同一個 hash slot:

    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => '{default}',
        'retry_after' => 90,
    ],

#### 其他的隊列驅動基本要求

針對列出的隊列驅動，必須安裝以下對應的相依套件：

<div class="content-list" markdown="1">
- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~3.0`
- Redis: `predis/predis ~1.0`
</div>

<a name="creating-jobs"></a>
## 建立任務

<a name="generating-job-classes"></a>
### 產生任務類別

應用程式所有可放入隊列中執行的任務都被存放在 `app/Jobs` 目錄。如果 `app/Jobs` 目錄不存在，執行 Artisan 指令 `make:job` 同時會建立該目錄，你可以使用 Artisan CLI 產生一個新的隊列任務：

    php artisan make:job ProcessPodcast

產生的任務類別會實作 `Illuminate\Contracts\Queue\ShouldQueue` 介面，意味著 Laravel 執行該任務時會將該任務類別以非同步的方式推送至隊列。

<a name="class-structure"></a>
### 類別結構

任務類別的結構非常簡單，通常會包含一個 `handle` 方法，該方法會在任務被隊列執行時呼叫。為了理解，讓我們看一下任務類別的範例。在這個範例中，我們假裝我們管理一個公開的推播服務，該服務需要在公開推播時處理上傳的播放檔案：

    <?php

    namespace App\Jobs;

    use App\Podcast;
    use App\AudioProcessor;
    use Illuminate\Bus\Queueable;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;

    class ProcessPodcast implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        protected $podcast;

        /**
         * 產生一個 Job 實例
         *
         * @param  Podcast  $podcast
         * @return void
         */
        public function __construct(Podcast $podcast)
        {
            $this->podcast = $podcast;
        }

        /**
         * 執行任務
         *
         * @param  AudioProcessor  $processor
         * @return void
         */
        public function handle(AudioProcessor $processor)
        {
            // 處理上傳的推播...
        }
    }

在這個範例中，我們能夠直接傳遞一個 [Eloquent 模型](/docs/{{version}}/eloquent) 至隊列任務的建構子。因為任務類別使用 `SerializesModels` trait，當任務被執行時， Eloquent 模型會優雅的被序列話化和解序列化。如果你的隊列任務在建構子接收一個 Eloquent 模型，只有模型的識別子(identifier)會被序列化被放進隊列中。當任務真正被處理時，隊列系統會自動的重新從資料庫獲取完整的模型實例。整個過程對於你的應用程式是完全透明的，避免在序列化整個 Eloquent 模型實例時出現問題。

`handle` 方法會在任務執行是被呼叫。注意我們能夠在 `handle` 方法傳遞的參數宣告依賴類別，Laravel 提供 [服務容器](/docs/{{version}}/container) 能夠自動的注入這些依賴類別。

> {note} 二進位資料，例如原始圖形內容，在傳遞至隊列任務時需要使用 `base64_encode` 函式進行傳遞。否則，該隊列任務在被放置進隊列時可能無法正確的序列化解析成 JSON 格式。

<a name="dispatching-jobs"></a>
## 執行任務 

當你撰寫完任務類別後，你可以呼叫類別內的 `dispatch` 方法執行任務。`dispatch` 方法的參數會被傳遞至任務類別的建構子中：

    <?php

    namespace App\Http\Controllers;

    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * 儲存一個新的推播
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // 建立推播...

            ProcessPodcast::dispatch($podcast);
        }
    }

<a name="delayed-dispatching"></a>
### 延遲執行

如果你想要延遲執行一個隊列任務，可以在執行隊列任務時使用 `delay` 方法。舉例來說，指定一個任務在十分鐘後執行：

    <?php

    namespace App\Http\Controllers;

    use Carbon\Carbon;
    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * 儲存新的推播
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // 新增推播 ...

            ProcessPodcast::dispatch($podcast)
                    ->delay(Carbon::now()->addMinutes(10));
        }
    }

> {note} The Amazon SQS queue service has a maximum delay time of 15 minutes.

<a name="job-chaining"></a>
### 隊列任務鏈

隊列任務鏈允許你指定一系列的隊列任務，並且依序的執行這些任務。如果隊列任務鏈中的其中一個工作失敗了，整個任務不會繼續被執行。你可以在任何被執行的隊列任務呼叫`withChain` 方法執行任務鏈：

    ProcessPodcast::withChain([
        new OptimizePodcast,
        new ReleasePodcast
    ])->dispatch();

<a name="customizing-the-queue-and-connection"></a>
### 自訂隊列 & 連結

#### 在特定的隊列中執行

透過推送並任務至不同的隊列，你可以對隊列任務進行「分類」，讓多個隊列任務之間分配不同的執行優先順序和 Worker 的執行數量。記住，這樣的行為並不會將對列任務推送到隊列設定檔內定義的「連線」。而只會將隊列任務在單一連線內推送到指定的隊列。在執行隊列任務時呼叫 `onQueue` 方法能指定隊列：

    <?php

    namespace App\Http\Controllers;

    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * 儲存一個新推播
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // 建立推播...

            ProcessPodcast::dispatch($podcast)->onQueue('processing');
        }
    }

#### 在特定的連線中執行

如果你正嘗試使用多個隊列連線，你可以指定要將對列任務推送到哪個連線。在執行隊列任務時呼叫 `onConnection` 方法能指定隊列任務推送至特定的連線：

    <?php

    namespace App\Http\Controllers;

    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * 儲存一個新推播
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // 建立推播...

            ProcessPodcast::dispatch($podcast)->onConnection('sqs');
        }
    }

當然,你可以串連 `onConnection` 和 `onQueue` 方法指定任務的連線和隊列：

    ProcessPodcast::dispatch($podcast)
                  ->onConnection('sqs')
                  ->onQueue('processing');

<a name="max-job-attempts-and-timeout"></a>
### 指定最大任務嘗試次數 / 逾時

#### 最大嘗試次數 

指定最大任務嘗試次數的一種方法是透過在執行 Artisan 指令時啟用 `--tries` 選項：

    php artisan queue:work --tries=3

不過，更精細的方式則是在任務類別內定義最大的嘗試次數。如果任務類別內指定了最大嘗試次數，在執行上述 Artisan 命令時類別內指定的最大次數優先於命令列指定的次數：

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * 隊列任務最大的嘗試次數
         *
         * @var int
         */
        public $tries = 5;
    }

<a name="time-based-attempts"></a>
#### 基於計時的嘗試

作為替代單純定義任務失敗時的最大嘗試次數的另一個方式，你可以定義任務的逾時時間，這讓任務可以在指定的時間內可以被重試無數次。在隊列任務類別內新增 `retryUntil` 方法來定義任務的最大逾時時間：

    /**
     * 預估該隊列任務多久會逾時
     *
     * @return \DateTime
     */
    public function retryUntil()
    {
        return now()->addSeconds(5);
    }

> {tip} 你也可以在你的隊列事件監聽類別內定義 `retryUntil` 方法。

#### 逾時

> {note} `timeout` 功能在 PHP 7.1+ 版本及 `pcntl` PHP 擴充元件均已優化。

同理，執行 Artisan 命令時使用 `--timeout` 選項能夠指定任務的最大秒數：

    php artisan queue:work --timeout=30

然而，你或許也想要在任務類別內定義任務允許被執行的最大秒數，如果在任務內指定了逾時，其優先權高於在 Artisan 命令列指定的秒數：

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * 任務在多久的時間內允許執行的最大秒數
         *
         * @var int
         */
        public $timeout = 120;
    }

<a name="rate-limiting"></a>
### 限制執行比例

> {note} 使用這個功能的前提是你的應用程式有根 [Redis 伺服器](/docs/{{version}}/redis) 進行互動

如果你的應用程式與 Redis 有所互動，你可以藉由時間或是併發次數限制隊列任務的執行。這個功能能夠協助你的隊列任務在與一些具備請求限制的 APIs 互動進行比例限制，舉例來說，使用 `throttle` 方法能夠限制該類型的任務只能在每 60 秒內執行 10 次。若無法鎖定，通常會將任務釋放回隊列以便稍後進行重試：

    Redis::throttle('key')->allow(10)->every(60)->then(function () {
        // 任務邏輯...
    }, function () {
        // 無法獲得鎖定...

        return $this->release(10);
    });

> {tip} 在上面的例子中， `key` 可能是任何的獨一無二的字串用來識別你想要限制執行比例的任務類型。比如，你可能會想要以類別名稱和對應操作 Eloquent 類別的 ID 作為鍵值。

另外，你可以指定同時處理隊列任務的 Worker 最大數量，這在限制隊列任務只能一次存取單一資源時特別有用。舉例來說，使用 `funnel` 方法能夠限制該類型的任務在同一時間內只能一次被單一個 Worker 處理：

    Redis::funnel('key')->limit(1)->then(function () {
        // 任務邏輯...
    }, function () {
        // 無法獲得鎖定...

        return $this->release(10);
    });

> {tip} 在限制執行比利時，很難得知任務真正所需的最大嘗試次數，因此，同時限制執行比例和[逾時嘗試](#time-based-attempts)十分有效。

<a name="error-handling"></a>
### 錯誤處理

如果執行隊列任務時一個例外狀況被拋出，該任務會自動的被釋放回隊列並且重新嘗試執行。該任務會在應用程式允許的最大嘗試次數內繼續地被執行和釋放回隊列，可藉由 `queue:work` Artisan 命令的 `--tries` 選項定義任務的最大嘗試次數。另外，也可以在任務類別內定義最大嘗試次數，更多關於執行隊列 Worker [請詳閱下列的資訊](#running-the-queue-worker)。

<a name="running-the-queue-worker"></a>
## Running The Queue Worker

Laravel includes a queue worker that will process new jobs as they are pushed onto the queue. You may run the worker using the `queue:work` Artisan command. Note that once the `queue:work` command has started, it will continue to run until it is manually stopped or you close your terminal:

    php artisan queue:work

> {tip} To keep the `queue:work` process running permanently in the background, you should use a process monitor such as [Supervisor](#supervisor-configuration) to ensure that the queue worker does not stop running.

Remember, queue workers are long-lived processes and store the booted application state in memory. As a result, they will not notice changes in your code base after they have been started. So, during your deployment process, be sure to [restart your queue workers](#queue-workers-and-deployment).

#### Processing A Single Job

The `--once` option may be used to instruct the worker to only process a single job from the queue:

    php artisan queue:work --once

#### Specifying The Connection & Queue

You may also specify which queue connection the worker should utilize. The connection name passed to the `work` command should correspond to one of the connections defined in your `config/queue.php` configuration file:

    php artisan queue:work redis

You may customize your queue worker even further by only processing particular queues for a given connection. For example, if all of your emails are processed in an `emails` queue on your `redis` queue connection, you may issue the following command to start a worker that only processes only that queue:

    php artisan queue:work redis --queue=emails

#### Resource Considerations

Daemon queue workers do not "reboot" the framework before processing each job. Therefore, you should free any heavy resources after each job completes. For example, if you are doing image manipulation with the GD library, you should free the memory with `imagedestroy` when you are done.

<a name="queue-priorities"></a>
### Queue Priorities

Sometimes you may wish to prioritize how your queues are processed. For example, in your `config/queue.php` you may set the default `queue` for your `redis` connection to `low`. However, occasionally you may wish to push a job to a `high` priority queue like so:

    dispatch((new Job)->onQueue('high'));

To start a worker that verifies that all of the `high` queue jobs are processed before continuing to any jobs on the `low` queue, pass a comma-delimited list of queue names to the `work` command:

    php artisan queue:work --queue=high,low

<a name="queue-workers-and-deployment"></a>
### Queue Workers & Deployment

Since queue workers are long-lived processes, they will not pick up changes to your code without being restarted. So, the simplest way to deploy an application using queue workers is to restart the workers during your deployment process. You may gracefully restart all of the workers by issuing the `queue:restart` command:

    php artisan queue:restart

This command will instruct all queue workers to gracefully "die" after they finish processing their current job so that no existing jobs are lost. Since the queue workers will die when the `queue:restart` command is executed, you should be running a process manager such as [Supervisor](#supervisor-configuration) to automatically restart the queue workers.

> {tip} The queue uses the [cache](/docs/{{version}}/cache) to store restart signals, so you should verify a cache driver is properly configured for your application before using this feature.

<a name="job-expirations-and-timeouts"></a>
### Job Expirations & Timeouts

#### Job Expiration

In your `config/queue.php` configuration file, each queue connection defines a `retry_after` option. This option specifies how many seconds the queue connection should wait before retrying a job that is being processed. For example, if the value of `retry_after` is set to `90`, the job will be released back onto the queue if it has been processing for 90 seconds without being deleted. Typically, you should set the `retry_after` value to the maximum number of seconds your jobs should reasonably take to complete processing.

> {note} The only queue connection which does not contain a `retry_after` value is Amazon SQS. SQS will retry the job based on the [Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) which is managed within the AWS console.

#### Worker Timeouts

The `queue:work` Artisan command exposes a `--timeout` option. The `--timeout` option specifies how long the Laravel queue master process will wait before killing off a child queue worker that is processing a job. Sometimes a child queue process can become "frozen" for various reasons, such as an external HTTP call that is not responding. The `--timeout` option removes frozen processes that have exceeded that specified time limit:

    php artisan queue:work --timeout=60

The `retry_after` configuration option and the `--timeout` CLI option are different, but work together to ensure that jobs are not lost and that jobs are only successfully processed once.

> {note} The `--timeout` value should always be at least several seconds shorter than your `retry_after` configuration value. This will ensure that a worker processing a given job is always killed before the job is retried. If your `--timeout` option is longer than your `retry_after` configuration value, your jobs may be processed twice.

#### Worker Sleep Duration

When jobs are available on the queue, the worker will keep processing jobs with no delay in between them. However, the `sleep` option determines how long the worker will "sleep" if there are no new jobs available. While sleeping, the worker will not process any new jobs - the jobs will be processed after the worker wakes up again.

    php artisan queue:work --sleep=3

<a name="supervisor-configuration"></a>
## 設定 Supervisor

#### 安裝 Supervisor

Supervisor 是一個 Linux 系統內的程序監看工具，同時能夠自動重啟失敗的 `queue:work` 程序。在 Ubuntu 發行版中你可以使用以下指令安裝 Supervisor ：

    sudo apt-get install supervisor

> {tip} 設定 Supervisor 聽起來很麻煩嗎？可以使用 [Laravel Forge](https://forge.laravel.com)，內建已經自動地為你的 Laravel 專安安裝並設定好 Supervisor。

#### Supervisor 設定檔

Supervisor 設定檔通常會位於 `/etc/supervisor/conf.d` 目錄。在這個目錄內，你可以建立不限數量的設定檔案來引導 Supervisor 如何監看你的程序。舉例來說，建立一個 `laravel-worker.conf` 檔案用於啟動並監看 `queue:work` 程序：

    [program:laravel-worker]
    process_name=%(program_name)s_%(process_num)02d
    command=php /home/forge/app.com/artisan queue:work sqs --sleep=3 --tries=3
    autostart=true
    autorestart=true
    user=forge
    numprocs=8
    redirect_stderr=true
    stdout_logfile=/home/forge/app.com/worker.log

在這個範例，`numprocs` 指令會引導 Supervisor 啟動並監看 8 個 `queue:work` 程序，並自動重啟失敗的程序。當然，你應該更改 `command` 選項內部分 `queue:work sqs` 設定，以達到你預期的隊列連線設定。

#### 啟動 Supervisor

一旦設定檔案被建立，你可以使用以下指令更新及啟動 Supervisor ：

    sudo supervisorctl reread

    sudo supervisorctl update

    sudo supervisorctl start laravel-worker:*

更多訊息，詳見 [Supervisor 參考文件](http://supervisord.org/index.html)。

<a name="dealing-with-failed-jobs"></a>
## 處理失敗的任務

有時候隊列中的任務執行失敗，別擔心，事情發生總有不如預期。Laravel 內建了方便的方法能夠指定隊列任務的最大嘗試次數。當一個隊列任務超過最大嘗試次數時，會新增至 `failed_jobs` 資料表。為了透過建立資料庫遷移產生 `failed_jobs` 資料表，你可以使用 `queue:failed-table` 指令：

    php artisan queue:failed-table

    php artisan migrate

接下來，執行 [隊列 worker](#running-the-queue-worker)，你應該在執行 `queue:work` 指令時指定 `--tries` 選項指定最大的錯誤嘗試次數。如果 `--tries` 選項沒有被指定，失敗的隊列任務會不斷的進行錯誤重試：

    php artisan queue:work redis --tries=3

<a name="cleaning-up-after-failed-jobs"></a>
### Cleaning Up After Failed Jobs

You may define a `failed` method directly on your job class, allowing you to perform job specific clean-up when a failure occurs. This is the perfect location to send an alert to your users or revert any actions performed by the job. The `Exception` that caused the job to fail will be passed to the `failed` method:

    <?php

    namespace App\Jobs;

    use Exception;
    use App\Podcast;
    use App\AudioProcessor;
    use Illuminate\Bus\Queueable;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class ProcessPodcast implements ShouldQueue
    {
        use InteractsWithQueue, Queueable, SerializesModels;

        protected $podcast;

        /**
         * Create a new job instance.
         *
         * @param  Podcast  $podcast
         * @return void
         */
        public function __construct(Podcast $podcast)
        {
            $this->podcast = $podcast;
        }

        /**
         * Execute the job.
         *
         * @param  AudioProcessor  $processor
         * @return void
         */
        public function handle(AudioProcessor $processor)
        {
            // Process uploaded podcast...
        }

        /**
         * The job failed to process.
         *
         * @param  Exception  $exception
         * @return void
         */
        public function failed(Exception $exception)
        {
            // Send user notification of failure, etc...
        }
    }

<a name="failed-job-events"></a>
### Failed Job Events

If you would like to register an event that will be called when a job fails, you may use the `Queue::failing` method. This event is a great opportunity to notify your team via email or [HipChat](https://www.hipchat.com). For example, we may attach a callback to this event from the `AppServiceProvider` that is included with Laravel:

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Queue;
    use Illuminate\Queue\Events\JobFailed;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Bootstrap any application services.
         *
         * @return void
         */
        public function boot()
        {
            Queue::failing(function (JobFailed $event) {
                // $event->connectionName
                // $event->job
                // $event->exception
            });
        }

        /**
         * Register the service provider.
         *
         * @return void
         */
        public function register()
        {
            //
        }
    }

<a name="retrying-failed-jobs"></a>
### Retrying Failed Jobs

To view all of your failed jobs that have been inserted into your `failed_jobs` database table, you may use the `queue:failed` Artisan command:

    php artisan queue:failed

The `queue:failed` command will list the job ID, connection, queue, and failure time. The job ID may be used to retry the failed job. For instance, to retry a failed job that has an ID of `5`, issue the following command:

    php artisan queue:retry 5

To retry all of your failed jobs, execute the `queue:retry` command and pass `all` as the ID:

    php artisan queue:retry all

If you would like to delete a failed job, you may use the `queue:forget` command:

    php artisan queue:forget 5

To delete all of your failed jobs, you may use the `queue:flush` command:

    php artisan queue:flush

<a name="job-events"></a>
## 任務事件

在 `Queue` [facade](/docs/{{version}}/facades) 使用 `before` 及 `after` 方法，你能指定回呼(callbacks)函式，在執行處理該隊列任務前或後執行對應的動作。這些回呼函式能夠完美的執行額外的事件記錄或是為儀表板提供統計資訊。通常會搭配 [service provider](/docs/{{version}}/providers) 用於呼叫這些方法。舉例來說，你可以使用 Laravel 內建的 `AppServiceProvider`:

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Queue;
    use Illuminate\Support\ServiceProvider;
    use Illuminate\Queue\Events\JobProcessed;
    use Illuminate\Queue\Events\JobProcessing;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 引導應用服務
         *
         * @return void
         */
        public function boot()
        {
            Queue::before(function (JobProcessing $event) {
                // $event->connectionName
                // $event->job
                // $event->job->payload()
            });

            Queue::after(function (JobProcessed $event) {
                // $event->connectionName
                // $event->job
                // $event->job->payload()
            });
        }

        /**
         * 註冊服務提供者
         *
         * @return void
         */
        public function register()
        {
            //
        }
    }

使用 `Queue` [facade](/docs/{{version}}/facades) 的 `looping` 方法，你能夠藉由定義回呼函式，在 worker 嘗試從隊列中獲取任務執行一些工作。舉例來說，你可以註冊一個閉包以還原前一個任務執行時拜留下的資料庫交易紀錄：

    Queue::looping(function () {
        while (DB::transactionLevel() > 0) {
            DB::rollBack();
        }
    });
