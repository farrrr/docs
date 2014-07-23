# 佇列

- [設定](#configuration)
- [基本用法](#basic-usage)
- [佇列閉包](#queueing-closures)
- [起動佇列監聽](#running-the-queue-listener)
- [常駐佇列工作](#daemon-queue-worker)
- [推送佇列](#push-queues)
- [失敗的工作](#failed-jobs)

<a name="configuration"></a>
## 設定

Laravel 佇列元件提供一個統一的API整合了許多不同的佇列服務，佇列允許你將一個執行任務延後執行，例如寄送郵件延後至你指定的時間，進而大幅的加快你的網站應用程式的速度。

佇列的設定檔在`app/config/queue.php`，在這個檔案你將可以找到框架中每種不同的佇列服務的連線設定，其中包含了[Beanstalkd](http://kr.github.com/beanstalkd), [IronMQ](http://iron.io), [Amazon SQS](http://aws.amazon.com/sqs), [Redis](http://redis.io),以及同步(本地端使用)驅動設定.

下列的composer.json設定可以依照你使用的佇列服務必需在使用前安裝：

- Beanstalkd: `pda/pheanstalk ~2.0`
- Amazon SQS: `aws/aws-sdk-php`
- IronMQ: `iron-io/iron_mq`

<a name="basic-usage"></a>
## 基本用法

#### 推送一個工作至佇列

要推送一個新的工作至佇列，請使用`Queue::push`方法：

	Queue::push('SendEmail', array('message' => $message));

#### 宣告一個工作處理程序

`push`方法的第一個參數為應該處理這個工作的類別方法名稱，第二個參數是一個要傳遞至處理程序的資料陣列，一個工作處理程序應該參照下列宣告：

	class SendEmail {

		public function fire($job, $data)
		{
			//
		}

	}

注意如果你未在第一個參數指定工作處理的類別及方法(如SendEmail@process)，那`fire`為類別中預設必需的方法用來接收被推送至佇列`Job`實例以及`data`。

#### 指定一個特定的處理程序方法

假如你想要用一個`fire`以外的方法處理工作，你可以在推送工作時指定方法如下：

	Queue::push('SendEmail@send', array('message' => $message));

#### 指定佇列使用的Queue

You may also specify the queue / tube a job should be sent to:

	Queue::push('SendEmail@send', array('message' => $message), 'emails');

#### Passing The Same Payload To Multiple Jobs

If you need to pass the same data to several queue jobs, you may use the `Queue::bulk` method:

	Queue::bulk(array('SendEmail', 'NotifyUser'), $payload);

#### Delaying The Execution Of A Job

Sometimes you may wish to delay the execution of a queued job. For instance, you may wish to queue a job that sends a customer an e-mail 15 minutes after sign-up. You can accomplish this using the `Queue::later` method:

	$date = Carbon::now()->addMinutes(15);

	Queue::later($date, 'SendEmail@send', array('message' => $message));

In this example, we're using the [Carbon](https://github.com/briannesbitt/Carbon) date library to specify the delay we wish to assign to the job. Alternatively, you may pass the number of seconds you wish to delay as an integer.

#### Deleting A Processed Job

Once you have processed a job, it must be deleted from the queue, which can be done via the `delete` method on the `Job` instance:

	public function fire($job, $data)
	{
		// Process the job...

		$job->delete();
	}

#### Releasing A Job Back Onto The Queue

If you wish to release a job back onto the queue, you may do so via the `release` method:

	public function fire($job, $data)
	{
		// Process the job...

		$job->release();
	}

You may also specify the number of seconds to wait before the job is released:

	$job->release(5);

#### Checking The Number Of Run Attempts

If an exception occurs while the job is being processed, it will automatically be released back onto the queue. You may check the number of attempts that have been made to run the job using the `attempts` method:

	if ($job->attempts() > 3)
	{
		//
	}

#### Accessing The Job ID

You may also access the job identifier:

	$job->getJobId();

<a name="queueing-closures"></a>
## Queueing Closures

You may also push a Closure onto the queue. This is very convenient for quick, simple tasks that need to be queued:

#### Pushing A Closure Onto The Queue

	Queue::push(function($job) use ($id)
	{
		Account::delete($id);

		$job->delete();
	});

> **Note:** Instead of making objects available to queued Closures via the `use` directive, consider passing primary keys and re-pulling the associated models from within your queue job. This often avoids unexpected serialization behavior.

When using Iron.io [push queues](#push-queues), you should take extra precaution queueing Closures. The end-point that receives your queue messages should check for a token to verify that the request is actually from Iron.io. For example, your push queue end-point should be something like: `https://yourapp.com/queue/receive?token=SecretToken`. You may then check the value of the secret token in your application before marshalling the queue request.

<a name="running-the-queue-listener"></a>
## Running The Queue Listener

Laravel includes an Artisan task that will run new jobs as they are pushed onto the queue. You may run this task using the `queue:listen` command:

#### Starting The Queue Listener

	php artisan queue:listen

You may also specify which queue connection the listener should utilize:

	php artisan queue:listen connection

Note that once this task has started, it will continue to run until it is manually stopped. You may use a process monitor such as [Supervisor](http://supervisord.org/) to ensure that the queue listener does not stop running.

You may pass a comma-delimited list of queue connections to the `listen` command to set queue priorities:

	php artisan queue:listen --queue=high,low

In this example, jobs on the `high-connection` will always be processed before moving onto jobs from the `low-connection`.

#### Specifying The Job Timeout Parameter

You may also set the length of time (in seconds) each job should be allowed to run:

	php artisan queue:listen --timeout=60

#### Specifying Queue Sleep Duration

In addition, you may specify the number of seconds to wait before polling for new jobs:

	php artisan queue:listen --sleep=5

Note that the queue only "sleeps" if no jobs are on the queue. If more jobs are available, the queue will continue to work them without sleeping.

#### Processing The First Job On The Queue

To process only the first job on the queue, you may use the `queue:work` command:

	php artisan queue:work

<a name="daemon-queue-worker"></a>
## Daemon Queue Worker

The `queue:work` also includes a `--daemon` option for forcing the queue worker to continue processing jobs without ever re-booting the framework. This results in a significant reduction of CPU usage when compared to the `queue:listen` command, but at the added complexity of needing to drain the queues of currently executing jobs during your deployments.

To start a queue worker in daemon mode, use the `--daemon` flag:

	php artisan queue:work connection --daemon

	php artisan queue:work connection --daemon --sleep=3

	php artisan queue:work connection --daemon --sleep=3 --tries=3

As you can see, the `queue:work` command supports most of the same options available to `queue:listen`. You may use the `php artisan help queue:work` command to view all of the available options.

### Deploying With Daemon Queue Workers

The simplest way to deploy an application using daemon queue workers is to put the application in maintenance mode at the beginning of your deploymnet. This can be done using the `php artisan down` command. Once the application is in maintenance mode, Laravel will not accept any new jobs off of the queue, but will continue to process existing jobs. Once enough time has passed for all of your existing jobs to execute (usually no longer than 30-60 seconds), you may stop the worker and continue your deployment process.

If you are using Supervisor or Laravel Forge, which utilizes Supervisor, you may typically stop a worker with a command like the following:

	supervisorctl stop worker-1

Once the queues have been drained and your fresh code has been deployed to your server, you should restart the daemon queue work. If you are using Supervisor, this can typically be done with a command like this:

	supervisorctl start worker-1

<a name="push-queues"></a>
## Push Queues

Push queues allow you to utilize the powerful Laravel 4 queue facilities without running any daemons or background listeners. Currently, push queues are only supported by the [Iron.io](http://iron.io) driver. Before getting started, create an Iron.io account, and add your Iron credentials to the `app/config/queue.php` configuration file.

#### Registering A Push Queue Subscriber

Next, you may use the `queue:subscribe` Artisan command to register a URL end-point that will receive newly pushed queue jobs:

	php artisan queue:subscribe queue_name http://foo.com/queue/receive

Now, when you login to your Iron dashboard, you will see your new push queue, as well as the subscribed URL. You may subscribe as many URLs as you wish to a given queue. Next, create a route for your `queue/receive` end-point and return the response from the `Queue::marshal` method:

	Route::post('queue/receive', function()
	{
		return Queue::marshal();
	});

The `marshal` method will take care of firing the correct job handler class. To fire jobs onto the push queue, just use the same `Queue::push` method used for conventional queues.

<a name="failed-jobs"></a>
## Failed Jobs

Since things don't always go as planned, sometimes your queued jobs will fail. Don't worry, it happens to the best of us! Laravel includes a convenient way to specify the maximum number of times a job should be attempted. After a job has exceeded this amount of attempts, it will be inserted into a `failed_jobs` table. The failed jobs table name can be configured via the `app/config/queue.php` configuration file.

To create a migration for the `failed_jobs` table, you may use the `queue:failed-table` command:

	php artisan queue:failed-table

You can specify the maximum number of times a job should be attempted using the `--tries` switch on the `queue:listen` command:

	php artisan queue:listen connection-name --tries=3

If you would like to register an event that will be called when a queue job fails, you may use the `Queue::failing` method. This event is a great opportunity to notify your team via e-mail or [HipChat](https://www.hipchat.com).

	Queue::failing(function($connection, $job, $data)
	{
		//
	});

To view all of your failed jobs, you may use the `queue:failed` Artisan command:

	php artisan queue:failed

The `queue:failed` command will list the job ID, connection, queue, and failure time. The job ID may be used to retry the failed job. For instance, to retry a failed job that has an ID of 5, the following command should be issued:

	php artisan queue:retry 5

If you would like to delete a failed job, you may use the `queue:forget` command:

	php artisan queue:forget 5

To delete all of your failed jobs, you may use the `queue:flush` command:

	php artisan queue:flush
