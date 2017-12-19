# 隊列

- [簡介](#introduction)
    - [連接 Vs. 隊列](#connections-vs-queues)
    - [隊列驅動基本要求](#driver-prerequisites)
- [建立任務](#creating-jobs)
    - [產生任務類別](#generating-job-classes)
    - [類別結構](#class-structure)
- [Dispatching Jobs](#dispatching-jobs)
    - [Delayed Dispatching](#delayed-dispatching)
    - [Job Chaining](#job-chaining)
    - [Customizing The Queue & Connection](#customizing-the-queue-and-connection)
    - [Specifying Max Job Attempts / Timeout Values](#max-job-attempts-and-timeout)
    - [Rate Limiting](#rate-limiting)
    - [Error Handling](#error-handling)
- [Running The Queue Worker](#running-the-queue-worker)
    - [Queue Priorities](#queue-priorities)
    - [Queue Workers & Deployment](#queue-workers-and-deployment)
    - [Job Expirations & Timeouts](#job-expirations-and-timeouts)
- [Supervisor Configuration](#supervisor-configuration)
- [Dealing With Failed Jobs](#dealing-with-failed-jobs)
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
## Dispatching Jobs

Once you have written your job class, you may dispatch it using the `dispatch` method on the job itself. The arguments passed to the `dispatch` method will be given to the job's constructor:

    <?php

    namespace App\Http\Controllers;

    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * Store a new podcast.
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // Create podcast...

            ProcessPodcast::dispatch($podcast);
        }
    }

<a name="delayed-dispatching"></a>
### Delayed Dispatching

If you would like to delay the execution of a queued job, you may use the `delay` method when dispatching a job. For example, let's specify that a job should not be available for processing until 10 minutes after it has been dispatched:

    <?php

    namespace App\Http\Controllers;

    use Carbon\Carbon;
    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * Store a new podcast.
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // Create podcast...

            ProcessPodcast::dispatch($podcast)
                    ->delay(Carbon::now()->addMinutes(10));
        }
    }

> {note} The Amazon SQS queue service has a maximum delay time of 15 minutes.

<a name="job-chaining"></a>
### Job Chaining

Job chaining allows you to specify a list of queued jobs that should be run in sequence. If one job in the sequence fails, the rest of the jobs will not be run. To execute a queued job chain, you may use the `withChain` method on any of your dispatchable jobs:

    ProcessPodcast::withChain([
        new OptimizePodcast,
        new ReleasePodcast
    ])->dispatch();

<a name="customizing-the-queue-and-connection"></a>
### Customizing The Queue & Connection

#### Dispatching To A Particular Queue

By pushing jobs to different queues, you may "categorize" your queued jobs and even prioritize how many workers you assign to various queues. Keep in mind, this does not push jobs to different queue "connections" as defined by your queue configuration file, but only to specific queues within a single connection. To specify the queue, use the `onQueue` method when dispatching the job:

    <?php

    namespace App\Http\Controllers;

    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * Store a new podcast.
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // Create podcast...

            ProcessPodcast::dispatch($podcast)->onQueue('processing');
        }
    }

#### Dispatching To A Particular Connection

If you are working with multiple queue connections, you may specify which connection to push a job to. To specify the connection, use the `onConnection` method when dispatching the job:

    <?php

    namespace App\Http\Controllers;

    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PodcastController extends Controller
    {
        /**
         * Store a new podcast.
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // Create podcast...

            ProcessPodcast::dispatch($podcast)->onConnection('sqs');
        }
    }

Of course, you may chain the `onConnection` and `onQueue` methods to specify the connection and the queue for a job:

    ProcessPodcast::dispatch($podcast)
                  ->onConnection('sqs')
                  ->onQueue('processing');

<a name="max-job-attempts-and-timeout"></a>
### Specifying Max Job Attempts / Timeout Values

#### Max Attempts

One approach to specifying the maximum number of times a job may be attempted is via the `--tries` switch on the Artisan command line:

    php artisan queue:work --tries=3

However, you may take a more granular approach by defining the maximum number of attempts on the job class itself. If the maximum number of attempts is specified on the job, it will take precedence over the value provided on the command line:

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * The number of times the job may be attempted.
         *
         * @var int
         */
        public $tries = 5;
    }

<a name="time-based-attempts"></a>
#### Time Based Attempts

As an alternative to defining how many times a job may be attempted before it fails, you may define a time at which the job should timeout. This allows a job to be attempted any number of times within a given time frame. To define the time at which a job should timeout, add a `retryUntil` method to your job class:

    /**
     * Determine the time at which the job should timeout.
     *
     * @return \DateTime
     */
    public function retryUntil()
    {
        return now()->addSeconds(5);
    }

> {tip} You may also define a `retryUntil` method on your queued event listeners.

#### Timeout

> {note} The `timeout` feature is optimized for PHP 7.1+ and the `pcntl` PHP extension.

Likewise, the maximum number of seconds that jobs can run may be specified using the `--timeout` switch on the Artisan command line:

    php artisan queue:work --timeout=30

However, you may also define the maximum number of seconds a job should be allowed to run on the job class itself. If the timeout is specified on the job, it will take precedence over any timeout specified on the command line:

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * The number of seconds the job can run before timing out.
         *
         * @var int
         */
        public $timeout = 120;
    }

<a name="rate-limiting"></a>
### Rate Limiting

> {note} This feature requires that your application can interact with a [Redis server](/docs/{{version}}/redis).

If your application interacts with Redis, you may throttle your queued jobs by time or concurrency. This feature can be of assistance when your queued jobs are interacting with APIs that are also rate limited. For example, using the `throttle` method, you may throttle a given type of job to only run 10 times every 60 seconds. If a lock can not be obtained, you should typically release the job back onto the queue so it can be retried later:

    Redis::throttle('key')->allow(10)->every(60)->then(function () {
        // Job logic...
    }, function () {
        // Could not obtain lock...

        return $this->release(10);
    });

> {tip} In the example above, the `key` may be any string that uniquely identifies the type of job you would like to rate limit. For example, you may wish to construct the key based on the class name of the job and the IDs of the Eloquent models it operates on.

Alternatively, you may specify the maximum number of workers that may simultaneously process a given job. This can be helpful when a queued job is modifying a resource that should only be modified by one job at a time. For example, using the `funnel` method, you may limit jobs of a given type to only be processed by one worker at a time:

    Redis::funnel('key')->limit(1)->then(function () {
        // Job logic...
    }, function () {
        // Could not obtain lock...

        return $this->release(10);
    });

> {tip} When using rate limiting, the number of attempts your job will need to run successfully can be hard to determine. Therefore, it is useful to combine rate limiting with [time based attempts](#time-based-attempts).

<a name="error-handling"></a>
### Error Handling

If an exception is thrown while the job is being processed, the job will automatically be released back onto the queue so it may be attempted again. The job will continue to be released until it has been attempted the maximum number of times allowed by your application. The maximum number of attempts is defined by the `--tries` switch used on the `queue:work` Artisan command. Alternatively, the maximum number of attempts may be defined on the job class itself. More information on running the queue worker [can be found below](#running-the-queue-worker).

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
## Supervisor Configuration

#### Installing Supervisor

Supervisor is a process monitor for the Linux operating system, and will automatically restart your `queue:work` process if it fails. To install Supervisor on Ubuntu, you may use the following command:

    sudo apt-get install supervisor

> {tip} If configuring Supervisor yourself sounds overwhelming, consider using [Laravel Forge](https://forge.laravel.com), which will automatically install and configure Supervisor for your Laravel projects.

#### Configuring Supervisor

Supervisor configuration files are typically stored in the `/etc/supervisor/conf.d` directory. Within this directory, you may create any number of configuration files that instruct supervisor how your processes should be monitored. For example, let's create a `laravel-worker.conf` file that starts and monitors a `queue:work` process:

    [program:laravel-worker]
    process_name=%(program_name)s_%(process_num)02d
    command=php /home/forge/app.com/artisan queue:work sqs --sleep=3 --tries=3
    autostart=true
    autorestart=true
    user=forge
    numprocs=8
    redirect_stderr=true
    stdout_logfile=/home/forge/app.com/worker.log

In this example, the `numprocs` directive will instruct Supervisor to run 8 `queue:work` processes and monitor all of them, automatically restarting them if they fail. Of course, you should change the `queue:work sqs` portion of the `command` directive to reflect your desired queue connection.

#### Starting Supervisor

Once the configuration file has been created, you may update the Supervisor configuration and start the processes using the following commands:

    sudo supervisorctl reread

    sudo supervisorctl update

    sudo supervisorctl start laravel-worker:*

For more information on Supervisor, consult the [Supervisor documentation](http://supervisord.org/index.html).

<a name="dealing-with-failed-jobs"></a>
## Dealing With Failed Jobs

Sometimes your queued jobs will fail. Don't worry, things don't always go as planned! Laravel includes a convenient way to specify the maximum number of times a job should be attempted. After a job has exceeded this amount of attempts, it will be inserted into the `failed_jobs` database table. To create a migration for the `failed_jobs` table, you may use the `queue:failed-table` command:

    php artisan queue:failed-table

    php artisan migrate

Then, when running your [queue worker](#running-the-queue-worker), you should specify the maximum number of times a job should be attempted using the `--tries` switch on the `queue:work` command. If you do not specify a value for the `--tries` option, jobs will be attempted indefinitely:

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
