# SSH

- [設定檔](#configuration)
- [基本用法](#basic-usage)
- [任務](#tasks)
- [SFTP 上傳](#sftp-uploads)
- [編輯遠端日誌](#tailing-remote-logs)

- [Configuration](#configuration)
- [Basic Usage](#basic-usage)
- [Tasks](#tasks)
- [SFTP Uploads](#sftp-uploads)
- [Tailing Remote Logs](#tailing-remote-logs)

<a name="configuration"></a>
## 設定檔

Laravel 可以簡單的 SSH 到遠端伺服器以及執行指令，讓你可以簡單的建立 Artisan 任務並且在遠端執行。`SSH` facade 提供了使用方式讓你連線到遠端伺服器並執行指令。

設定檔位在 `app/config/remote.php` ，包含所有你需要設定的遠端連線設定， `connections` 陣列裡有以遠端伺服器名稱作為鍵值的列表。只要在 `connections` 陣列設定好認證，你就準備好可以執行遠端任務了。記得 `SSH` 可以經由密碼或 SSH key認證。

## Configuration

Laravel includes a simple way to SSH into remote servers and run commands, allowing you to easily build Artisan tasks that work on remote servers. The `SSH` facade provides the access point to connecting to your remote servers and running commands.

The configuration file is located at `app/config/remote.php`, and contains all of the options you need to configure your remote connections. The `connections` array contains a list of your servers keyed by name. Simple populate the credentials in the `connections` array and you will be ready to start running remote tasks. Note that the `SSH` can authenticate using either a password or an SSH key.

<a name="basic-usage"></a>
## 基本用法

**在預設伺服器執行指令**

使用 `SSH::run` 方法，再預設的遠端伺服器執行指令：

## Basic Usage

**Running Commands On The Default Server**

To run commands on your `default` remote connection, use the `SSH::run` method:

	SSH::run(array(
		'cd /var/www',
		'git pull origin master',
	));


**在特定伺服器執行指令**

此外，你可以使用 `into` 方法在特定的伺服器上執行指令：

**Running Commands On A Specific Connection**

Alternatively, you may run commands on a specific connection using the `into` method:

	SSH::into('staging')->run(array(
		'cd /var/www',
		'git pull origin master',
	));

**捕捉指令的輸出**

你可以經由傳遞閉合函數到 `run` 方法，捕捉所有遠端指令的即時輸出：

**Catching Output From Commands**

You may catch the "live" output of your remote commands by passing a Closure into the `run` method:

	SSH::run($commands, function($line)
	{
		echo $line.PHP_EOL;
	});

## 任務

## Tasks
<a name="tasks"></a>

如果你需要定義一組每次都會執行的指令，你可以用 `define` 方法定義一個「任務」：

If you need to define a group of commands that should always be run together, you may use the `define` method to define a `task`:

	SSH::into('staging')->define('deploy', array(
		'cd /var/www',
		'git pull origin master',
		'php artisan migrate',
	));

你可以用 `task` 方法執行定義過的任務：

Once the task has been defined, you may use the `task` method to run it:

	SSH::into('staging')->task('deploy', function($line)
	{
		echo $line.PHP_EOL;
	});

<a name="sftp-uploads"></a>
## SFTP 上傳

`SSH` 類別也有簡單的方法可以上傳檔案，或甚至是字串到遠端伺服器。使用 `put` and `putString` 方法：

## SFTP Uploads

The `SSH` class also includes a simple way to upload files, or even strings, to the server using the `put` and `putString` methods:

	SSH::into('staging')->put($localFile, $remotePath);

	SSH::into('staging')->putString('Foo', $remotePath);

<a name="tailing-remote-logs"></a>
## 編輯遠端日誌

Laravel 有一個有用的指令可以讓你在附加日誌內容在任何遠端伺服器的 `laravel.log` 尾端。使用 Artisan 的 `tail` 指令以及指定遠端連線的伺服器名稱： 

## Tailing Remote Logs

Laravel includes a helpful command for tailing the `laravel.log` files on any of your remote connections. Simple use the `tail` Artisan command and specify the name of the remote connection you would like to tail:

	php artisan tail staging

	php artisan tail staging --path=/path/to/log.file