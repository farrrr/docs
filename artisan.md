# Artisan 指令列

- [介紹](#introduction)
- [撰寫指令](#writing-commands)
    - [產生指令](#generating-commands)
    - [指令結構](#command-structure)
    - [閉包指令](#closure-commands)
- [定義預期的輸入](#defining-input-expectations)
    - [參數](#arguments)
    - [選項](#options)
    - [輸入陣列](#input-arrays)
    - [輸入說明](#input-descriptions)
- [指令 I/O](#command-io)
    - [取得輸入](#retrieving-input)
    - [互動式輸入](#prompting-for-input)
    - [自訂輸出](#writing-output)
- [註冊指令](#registering-commands)
- [使用程式碼呼叫指令](#programmatically-executing-commands)
    - [在指令中呼叫其他指令](#calling-commands-from-other-commands)

<a name="introduction"></a>
## 介紹

Artisan 是 Laravel 內建的指令集合，它能提供許多好用的指令來協助你開發程式。你可以使用 `list` 查詢更多指令：

    php artisan list

每個指令都有輔助說明，會告訴你有哪些參數及選項可以用。在需要查詢的指令前加上 `help` 即可顯示輔助說明內容:

    php artisan help migrate

#### Laravel REPL

所有 Laravel 的應用程式都可以使用 Tinker，是基於 [PsySH](https://github.com/bobthecow/psysh) 這個套件所提供的 REPL。 Tinker 可以直接操控你的整個 Laravel 應用程式，包括 Eloquent ORM、任務、事件等。 執行 `tinker` 這個指令，即可進入 Tinker 環境：

    php artisan tinker

<a name="writing-commands"></a>
## 撰寫指令

除了 Laravel 提供的原生指令外，你也可以自訂指令。預設檔案路徑是在 `app/Console/Commands`。然而，只要指令可以被 Composer 載入，那你就可以任意的選擇檔案路徑。

<a name="generating-commands"></a>
### 產生指令

要產生一個新指令，請使用 `make:command`。該指令會在 `app/Console/Commands` 這個目錄中建立檔案。如果你的 Laravel 應用程式中沒有這個目錄，別擔心！當你第一次使用 `make:command` 時，會即時建立該目錄。產生的指令會包括所有指令中預設的屬性與方法：

    php artisan make:command SendEmails

<a name="command-structure"></a>
### 指令結構

產生新的指令後，應該先宣告 `signature` 和 `description` 的屬性內容，這會在使用 `list` 這個指令的時候顯示出來。 當指令被執行時， `handle` 方法會被呼叫，因此你可以將任何的指令邏輯放到該方法中。

> {tip} 為了讓程式碼更有效地使用（重用性），最好讓終端指令的程式碼保持輕量化，並讓它們緩載到應用程式服務的任務完成。在下列範例中，請注意！我們注入了一個服務類別來完成發送信件的「重任」。

讓我們看一個例子。請注意，我們可以在建構子中注入任何需要的依賴， Laravel 的 [服務容器](/docs/{{version}}/container) 將會自動注入任何型別提示的依賴到建構子中。

    <?php

    namespace App\Console\Commands;

    use App\User;
    use App\DripEmailer;
    use Illuminate\Console\Command;

    class SendEmails extends Command
    {
        /**
         * 指令列的名稱及用法
         *
         * @var string
         */
        protected $signature = 'email:send {user}';

        /**
         * 指令列的描述
         *
         * @var string
         */
        protected $description = 'Send drip e-mails to a user';

        /**
         * The drip e-mail service.
         *
         * @var DripEmailer
         */
        protected $drip;

        /**
         * 建立新的指令實例
         *
         * @param  DripEmailer  $drip
         * @return void
         */
        public function __construct(DripEmailer $drip)
        {
            parent::__construct();

            $this->drip = $drip;
        }

        /**
         * 執行指令
         *
         * @return mixed
         */
        public function handle()
        {
            $this->drip->send(User::find($this->argument('user')));
        }
    }

<a name="closure-commands"></a>
### 閉包指令

基於閉包的指令提供了有別於使用類別定義終端指令的方法。簡單的說，路由閉包是另一種撰寫指令的方式。在 `app/Console/Kernel.php` 檔案的 `commands` 這個方法中， Laravel 會載入 `routes/console.php` 這個檔案：

    /**
     * 為應用程式註冊基於閉包的指令
     *
     * @return void
     */
    protected function commands()
    {
        require base_path('routes/console.php');
    }

即使這個檔案沒有定義 HTTP 路由，它仍可以透過路由終端定義到應用程式中。在這個檔案中，你可以使用 `Artisan::command` 這個方法定義所有基於閉包的路由。 `command` 方法可以接受兩個參數： 其一是[命名](#defining-input-expectations) ，另一個取得指令參數與選項的閉包：

    Artisan::command('build {project}', function ($project) {
        $this->info("Building {$project}!");
    });

因為閉包綁定最底層的指令實例，所以你完全可以使用指令類別的所有輔助方法

#### 型態提示依賴

除了接收指令參數與選項外，指令閉包還可以使用型別提示從 [服務容器](/docs/{{version}}/container) 中注入所需的任何依賴：

    use App\User;
    use App\DripEmailer;

    Artisan::command('email:send {user}', function (DripEmailer $drip, $user) {
        $drip->send(User::find($user));
    });

#### 撰寫閉包指令的描述

當定義一個基於閉包的指令時，你可以使用 `describe` 方法來新增指令的描述。這個描述會在你執行 `php artisan list` 或 `php artisan help` 時顯示：

    Artisan::command('build {project}', function ($project) {
        $this->info("Building {$project}!");
    })->describe('Build the project');

<a name="defining-input-expectations"></a>
## 定義預期的輸入

在撰寫終端指令時，通常還會透過參數或選項來取得使用者輸入的指令。 Laravel 可以非常方便的使用 `signature` 屬性來定義你預期用戶輸入的內容。 `signature` 屬性可以給你使用單一且可讀性高，還有類似路由的語法來定義名稱、參數和選項。

<a name="arguments"></a>
### 參數

所有使用者輸入的參數與選項都會在大括號中。在接下來的範例中，這個指令定義了一個 **必要的** 參數： `user`:

    /**
     * 指令列的命名和用法
     *
     * @var string
     */
    protected $signature = 'email:send {user}';

你也可以建立可選參數，並定義參數的預設值：

    // 可選參數...
    email:send {user?}

    // 帶有預設值的可選參數...
    email:send {user=foo}

<a name="options"></a>
### 選項

選項，很像參數，是用戶輸入的另一種方式。當指令列指定選項時，它們以兩個字符（`--`）作為前綴。有兩種類型的選項：可接受值和不可接受值。不接受值的選項又可作為布林值的「開關」。讓我們看一下這種類型選項的例子：

    /**
     * 指令列的命名和用法
     *
     * @var string
     */
    protected $signature = 'email:send {user} {--queue}';

在這個例子中，可以在執行 Artisan 指令中加入 `--queue`，如果有輸入 `--queue` ，那麼將會回傳 `true` 。除此之外，則回傳 `false` ：

    php artisan email:send 1 --queue

<a name="options-with-values"></a>
#### 附值的選項

接下來，如果需要使用者輸入參數的值，那麼只需要在選項名稱後面添加 `=`：

    /**
     * 指令列的命名和用法
     *
     * @var string
     */
    protected $signature = 'email:send {user} {--queue=}';

在這個例子中，使用者可以為這個選項輸入想要的值：

    php artisan email:send 1 --queue=default

你可以指定預設值給選項：

    email:send {user} {--queue=default}

<a name="option-shortcuts"></a>
#### 選項的簡寫

你只需要簡單的在選項名稱之前指定簡寫，並使用 `|` 分隔符號將它與完整的選項名稱隔開：

    email:send {user} {--Q|queue}

<a name="input-arrays"></a>
### 輸入陣列

如果你想定義預期輸入的參數或選項為陣列，你可以使用 `*` 符號。首先，讓我們先看一下一個陣列參數的實例：

    email:send {user*}

當呼叫該方法時，可以使用 `user` 參數給指令列。例如，以下的指令會設置 `user` 為 `['foo', 'bar']`:

    php artisan email:send foo bar

當定義期望輸入的參數或選項的值為陣列時，應在要個別輸入選項值的前綴:

    email:send {user} {--id=*}

    php artisan email:send --id=1 --id=2

<a name="input-descriptions"></a>
### 輸入說明

你可以透過冒號為輸入的參數和選項個別說明如何使用。如果你需要一點額外的空間來定義你的指令，可以隨意撰寫多行：

    /**
     * 命令列的命名和用法
     *
     * @var string
     */
    protected $signature = 'email:send
                            {user : The ID of the user}
                            {--queue= : Whether the job should be queued}';

<a name="command-io"></a>
## 指令 I/O

<a name="retrieving-input"></a>
### 取得輸入

While your command is executing, you will obviously need to access the values for the arguments and options accepted by your command. To do so, you may use the `argument` and `option` methods:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $userId = $this->argument('user');

        //
    }

If you need to retrieve all of the arguments as an `array`, call the `arguments` method:

    $arguments = $this->arguments();

Options may be retrieved just as easily as arguments using the `option` method. To retrieve all of the options as an array, call the `options` method:

    // Retrieve a specific option...
    $queueName = $this->option('queue');

    // Retrieve all options...
    $options = $this->options();

If the argument or option does not exist, `null` will be returned.

<a name="prompting-for-input"></a>
### 互動式輸入

In addition to displaying output, you may also ask the user to provide input during the execution of your command. The `ask` method will prompt the user with the given question, accept their input, and then return the user's input back to your command:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $name = $this->ask('What is your name?');
    }

The `secret` method is similar to `ask`, but the user's input will not be visible to them as they type in the console. This method is useful when asking for sensitive information such as a password:

    $password = $this->secret('What is the password?');

#### Asking For Confirmation

If you need to ask the user for a simple confirmation, you may use the `confirm` method. By default, this method will return `false`. However, if the user enters `y` or `yes` in response to the prompt, the method will return `true`.

    if ($this->confirm('Do you wish to continue?')) {
        //
    }

#### Auto-Completion

The `anticipate` method can be used to provide auto-completion for possible choices. The user can still choose any answer, regardless of the auto-completion hints:

    $name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);

#### Multiple Choice Questions

If you need to give the user a predefined set of choices, you may use the `choice` method. You may set the default value to be returned if no option is chosen:

    $name = $this->choice('What is your name?', ['Taylor', 'Dayle'], $default);

<a name="writing-output"></a>
### 自訂輸出

To send output to the console, use the `line`, `info`, `comment`, `question` and `error` methods. Each of these methods will use appropriate ANSI colors for their purpose. For example, let's display some general information to the user. Typically, the `info` method will display in the console as green text:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $this->info('Display this on the screen');
    }

To display an error message, use the `error` method. Error message text is typically displayed in red:

    $this->error('Something went wrong!');

If you would like to display plain, uncolored console output, use the `line` method:

    $this->line('Display this on the screen');

#### Table Layouts

The `table` method makes it easy to correctly format multiple rows / columns of data. Just pass in the headers and rows to the method. The width and height will be dynamically calculated based on the given data:

    $headers = ['Name', 'Email'];

    $users = App\User::all(['name', 'email'])->toArray();

    $this->table($headers, $users);

#### Progress Bars

For long running tasks, it could be helpful to show a progress indicator. Using the output object, we can start, advance and stop the Progress Bar. First, define the total number of steps the process will iterate through. Then, advance the Progress Bar after processing each item:

    $users = App\User::all();

    $bar = $this->output->createProgressBar(count($users));

    foreach ($users as $user) {
        $this->performTask($user);

        $bar->advance();
    }

    $bar->finish();

For more advanced options, check out the [Symfony Progress Bar component documentation](https://symfony.com/doc/2.7/components/console/helpers/progressbar.html).

<a name="registering-commands"></a>
## 註冊指令

Because of the `load` method call in your console kernel's `commands` method, all commands within the `app/Console/Commands` directory will automatically be registered with Artisan. In fact, you are free to make additional calls to the `load` method to scan other directories for Artisan commands:

    /**
     * Register the commands for the application.
     *
     * @return void
     */
    protected function commands()
    {
        $this->load(__DIR__.'/Commands');
        $this->load(__DIR__.'/MoreCommands');

        // ...
    }

You may also manually register commands by adding its class name to the `$command` property of your `app/Console/Kernel.php` file. When Artisan boots, all the commands listed in this property will be resolved by the [service container](/docs/{{version}}/container) and registered with Artisan:

    protected $commands = [
        Commands\SendEmails::class
    ];

<a name="programmatically-executing-commands"></a>
## 使用程式碼呼叫指令

Sometimes you may wish to execute an Artisan command outside of the CLI. For example, you may wish to fire an Artisan command from a route or controller. You may use the `call` method on the `Artisan` facade to accomplish this. The `call` method accepts the name of the command as the first argument, and an array of command parameters as the second argument. The exit code will be returned:

    Route::get('/foo', function () {
        $exitCode = Artisan::call('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    });

Using the `queue` method on the `Artisan` facade, you may even queue Artisan commands so they are processed in the background by your [queue workers](/docs/{{version}}/queues). Before using this method, make sure you have configured your queue and are running a queue listener:

    Route::get('/foo', function () {
        Artisan::queue('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    });

#### Passing Array Values

If your command defines an option that accepts an array, you may simply pass an array of values to that option:

    Route::get('/foo', function () {
        $exitCode = Artisan::call('email:send', [
            'user' => 1, '--id' => [5, 13]
        ]);
    });

#### Passing Boolean Values

If you need to specify the value of an option that does not accept string values, such as the `--force` flag on the `migrate:refresh` command, you should pass `true` or `false`:

    $exitCode = Artisan::call('migrate:refresh', [
        '--force' => true,
    ]);

<a name="calling-commands-from-other-commands"></a>
### 在指令中呼叫其他指令

Sometimes you may wish to call other commands from an existing Artisan command. You may do so using the `call` method. This `call` method accepts the command name and an array of command parameters:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $this->call('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    }

If you would like to call another console command and suppress all of its output, you may use the `callSilent` method. The `callSilent` method has the same signature as the `call` method:

    $this->callSilent('email:send', [
        'user' => 1, '--queue' => 'default'
    ]);
