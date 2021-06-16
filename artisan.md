# Artisan Console

- [介紹](#introduction)
    - [Tinker (REPL)](#tinker)
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
- [使用程式碼執行指令](#programmatically-executing-commands)
    - [在指令中呼叫其他指令](#calling-commands-from-other-commands)

<a name="introduction"></a>
## 介紹

Artisan 是 Laravel 內建的指令集合，它能提供許多好用的指令來協助你開發程式。你可以使用 `list` 查詢所有可用的 Artisan 指令列表：

    php artisan list

每個指令還包含一個幫助畫面，可以顯示和描述該指令所有可用的參數和選項，要查看幫助畫面的話，請在指令名稱前加上 `help`

    php artisan help migrate

<a name="tinker"></a>
### Tinker (REPL)

Laravel Tinker 是一個由 [PsySH](https://github.com/bobthecow/psysh) 套件提供給 Laravel 框架的強大 REPL。

#### 安裝

所有的 Laravel 應用程式都預設包含 Tinker，但如果需要的話，你可以藉由 Composer 手動安裝：

    composer require laravel/tinker

#### 使用方式

Tinker 允許你在指令列上和整個 Laravel 應用程式互動，包含 Eloquent ORM、任務、事件等，想進入 Tinker 環境，請執行 `tinker` Artisan 指令：

    php artisan tinker

你可以使用 `vendor:publish` 指令發布 Tinker 的設定檔：

    php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"

> {note} `Dispatchable` 類別上的 `dispatch` 輔助函式和 `dispatch` 方法依賴記憶體回收機制來把任務放在隊列上。因此，在使用 tinker 的時候，你應該使用 `Bus::dispatch` 或 `queue::push` 來觸發任務。

#### 指令白名單

Tinker 利用白名單來確認哪些 Artisan 指令能在 shell 上執行，預設情況下，你可以執行 `clear-compiled`、`down`、`env`、`inspire`、`migrate`、`optimize` 和 `up` 指令。如果你想要將更多指令列入白名單，你可以在 `tinker.php` 設定檔的 `commands` 陣列中加入它們：

    'commands' => [
        // App\Console\Commands\ExampleCommand::class,
    ],

#### 黑名單別名

一般來說，Tinker 會根據你的 Tinker 中的需求自動為類別增加別名，但你可能不希望某些類別被加上別名，你可以在 `tinker.php` 設定檔中的 `dont_alias` 陣列中列舉這些類別來達成這種操作：

    'dont_alias' => [
        App\User::class,
    ],

<a name="writing-commands"></a>
## 撰寫指令

除了 Artisan 提供的指令以外，你還可以編寫自定義的指令。指令一般儲存在 `app/Console/Commands` 資料夾中，但只要 Composer 可以加載，你可以自由選擇儲存的位置。

<a name="generating-commands"></a>
### 產生指令

可以使用 `make:command` 來產生一個新的 Artisan 指令。這個指令會在 `app/Console/Commands` 資料夾中產生一個新的 command 類別。如果你的應用程式中沒有此資料夾，不用擔心，它會在你第一次執行 `make:command` 時被產生，產生的指令會包含所有指令中預設存在的屬性和方法：

    php artisan make:command SendEmails

<a name="command-structure"></a>
### 指令結構

產生你的指令之後，你應該填寫類別的 `signature` 和 `description` 他們會在你使用 `list` 指令時顯示。`handle` 方法會在你呼叫指令時被執行，你可以將你的指令邏輯放在這個方法。

> {tip} 為了更好的程式碼複用，最好讓終端指令保持輕量，並讓它們延遲到應用程式服務中完成。在下面的例子中，我們將注入一個服務類別來完成發送電子郵件這種「繁重的任務」。

讓我們來看一個指令的例子，我們可以在指令的 `handle` 方法中注入任何需要的依賴。Laravel 的 [服務容器](/docs/{{version}}/container) 將會自動注入所有在此方法中帶有型別提示的依賴：

    <?php

    namespace App\Console\Commands;

    use App\DripEmailer;
    use App\User;
    use Illuminate\Console\Command;

    class SendEmails extends Command
    {
        /**
         * The name and signature of the console command.
         *
         * @var string
         */
        protected $signature = 'email:send {user}';

        /**
         * The console command description.
         *
         * @var string
         */
        protected $description = 'Send drip e-mails to a user';

        /**
         * Create a new command instance.
         *
         * @return void
         */
        public function __construct()
        {
            parent::__construct();
        }

        /**
         * Execute the console command.
         *
         * @param  \App\DripEmailer  $drip
         * @return mixed
         */
        public function handle(DripEmailer $drip)
        {
            $drip->send(User::find($this->argument('user')));
        }
    }

<a name="closure-commands"></a>
### 閉包指令

基於閉包的指令提供了有別於使用類別定義終端指令的方法。就像路由閉包是控制器的替代方法一樣，閉包指令可以視為是指令類別的替代方法。在 `app/Console/Kernel.php` 檔案中的 `commands` 方法，Laravel 載入了 `routes/console.php` 檔案：

    /**
     * Register the Closure based commands for the application.
     *
     * @return void
     */
    protected function commands()
    {
        require base_path('routes/console.php');
    }

雖然這個檔案沒有定義 HTTP 路由，但它定義了基於終端的應用程式入口(路由)。在這個檔案中，你可以使用`Artisan::command` 方法定義所有閉包路由。`command` 方法接受兩個參數：[指令簽章](#defining-input-expectations)和一個接收指令參數和選項的閉包：

    Artisan::command('build {project}', function ($project) {
        $this->info("Building {$project}!");
    });

因為閉包綁定底層的指令實例，所以你可以使用所有在指令類別中可以使用的輔助函式。

#### 型別提示依賴

除了接收指令的參數和選項以外，指令閉包也可以使用型別提示從[服務容器](/docs/{{version}}/container)中解析其他的依賴：

    use App\DripEmailer;
    use App\User;

    Artisan::command('email:send {user}', function (DripEmailer $drip, $user) {
        $drip->send(User::find($user));
    });

#### 閉包指令描述

當定義一個基於指令的閉包時，你可以使用 `describe` 方法來為你的指令增加描述。這個描述會在你執行 `php artisan list` 或 `php artisan help` 指令時顯示：

    Artisan::command('build {project}', function ($project) {
        $this->info("Building {$project}!");
    })->describe('Build the project');

<a name="defining-input-expectations"></a>
## 定義預期的輸入

在撰寫終端指令時，透過參數或選項來取得使用者的輸入是很常見的，Laravel 可以非常方便的在指令中的 `signature` 屬性定義你期望使用者輸入的內容。`signature` 屬性允許你用單一的、可讀性高且類似路由的語法來定義名稱、參數和選項。

<a name="arguments"></a>
### 參數

所以使用者輸入的參數和選項都會在大括號中，在接下來的範例中，這個指令定義了一個**必要的**參數： `user`：

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'email:send {user}';

你也可以建立可選的參數，並定義參數的預設值：

    // Optional argument...
    email:send {user?}

    // Optional argument with default value...
    email:send {user=foo}

<a name="options"></a>
### 選項

選項類似參數，是另一種使用者輸入的格式。當指令列指定選項時會以兩個字符 (`--`) 做為前綴。有兩種類型的選項：可接受值與不可接受值，不接受值的選項可作為布林值的「開關」，讓我們看看這種類型選項的例子：

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'email:send {user} {--queue}';

在這個範例中，可以在呼叫 Artisan 指令時指定 `--queue` 開關，如果有輸入 `--queue`，選項的值會被設為 `true`，沒有的話則會被設為 `false`：

    php artisan email:send 1 --queue

<a name="options-with-values"></a>
#### 附值的選項

接下來讓我們看看一個附值的選項，如果使用者要為選項指定值，需要在選項名稱後面加上 `=` 符號：

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'email:send {user} {--queue=}';

在這個範例中，使用者可以像這樣輸入選項的值：

    php artisan email:send 1 --queue=default

你可以在選項名稱後指定預設值，如果使用者沒有輸入選項的值，將會使用預設值：

    email:send {user} {--queue=default}

<a name="option-shortcuts"></a>
#### 選項簡寫

要在定義選項時指定簡寫，你可以在選項名稱前指定它，並使用 | 分隔符號和完整的名稱隔開：

    email:send {user} {--Q|queue}

<a name="input-arrays"></a>
### 輸入陣列

如果在定義參數或選項時預期使用者輸入陣列，你可以使用 `*` 符號，首先讓我們看看一個陣列參數的範例：

    email:send {user*}

當呼叫這個方法時，可以把 `user` 參數傳給指令列，比如說，以下的指令會把 `user` 的值設為 `['foo', 'bar']`：

    php artisan email:send foo bar

當定義一個預期輸入陣列的參數時，傳給指令的每個選項值都應該加上選項名稱的前綴：

    email:send {user} {--id=*}

    php artisan email:send --id=1 --id=2

<a name="input-descriptions"></a>
### 輸入說明

你可以透過冒號為輸入的參數和選項個別說明如何使用，如果你需要一些額外的空間來定義你的指令，可以隨意分開成多行：

    /**
     * The name and signature of the console command.
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

在指令執行時，顯然你需要存取指令接受的參數和選項，你可以使用 `argument` 和 `option` 方法來做到：

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

如果你需要以 `array` 的方式取得所有的參數，呼叫 `arguments` 方法：

    $arguments = $this->arguments();

藉由 `option` 方法，你可以像取得參數一樣輕鬆的取得選項。可以呼叫 `options` 方法以陣列的方式取得所有選項：

    // Retrieve a specific option...
    $queueName = $this->option('queue');

    // Retrieve all options...
    $options = $this->options();

如果參數或選項不存在，將會回傳 `null`。

<a name="prompting-for-input"></a>
### 互動式輸入

除了顯示輸出以外，你還可以要求使用者在執行指令時提供輸入。`ask` 方法將用給定的問題提示使用者，接受輸入，並將使用者的輸入回傳到你的指令：

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $name = $this->ask('What is your name?');
    }

`secret` 方法類似 `ask`，但使用者的輸入並不會顯示在終端上，這個方法很適合用在要求使用者輸入密碼之類的敏感資訊：

    $password = $this->secret('What is the password?');

#### 請求確認

如果你需要對使用者進行簡單的確認，你可以使用 `confirm` 方法。在預設情況，這個方法會回傳 `false`，但如果使用者在提示中輸入 `y` 或 `yes` 則會回傳 `true`。

    if ($this->confirm('Do you wish to continue?')) {
        //
    }

#### 自動補全

`anticipate` 方法可以用來提供自動補全可能的選項，無論自動補全是否提示，使用者仍然可以選擇任何答案：

    $name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);

或者你可以將閉包做為第二個參數傳到 `anticipate` 方法，這個閉包會在每一次使用者輸入字元時被呼叫，它應該接受一個包含使用者輸入的字串參數，並回傳一個用於自動補全的選項陣列：

    $name = $this->anticipate('What is your name?', function ($input) {
        // Return auto-completion options...
    });

#### 選擇題

如果你需要給使用者預先定義好的選項，你可以使用 `choice` 方法。你還可以設置一個陣列的索引，它會在沒有選項被選擇時被當作預設值：

    $name = $this->choice('What is your name?', ['Taylor', 'Dayle'], $defaultIndex);

除此之外，`choice` 方法接受可選的第四和第五個參數，用於確認選擇有效回應的最大嘗試次數以及是否允許多選：

    $name = $this->choice(
        'What is your name?',
        ['Taylor', 'Dayle'],
        $defaultIndex,
        $maxAttempts = null,
        $allowMultipleSelections = false
    );

<a name="writing-output"></a>
### 自訂輸出

使用 `line`、`info`、`comment`、`question` 和 `error` 方法來傳送輸出到終端，每個方法都會使用適合的 ANSI 顏色來表達他們的目的。例如，讓我們向使用者顯示一些一般資訊，通常會使用 `info` 方法，它會在終端中顯示綠色的文字：

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $this->info('Display this on the screen');
    }

要顯示錯誤訊息，可以使用 `error` 方法，錯誤訊息一般以紅色顯示：

    $this->error('Something went wrong!');

如果你想顯示沒有顏色的終端輸出，可以使用 `line` 方法：

    $this->line('Display this on the screen');

#### 表格佈局

`table` 方法讓更輕鬆地正確格式化多行/列的資料，只需要傳送標題和行，它會根據給定的資料動態計算出長寬：

    $headers = ['Name', 'Email'];

    $users = App\User::all(['name', 'email'])->toArray();

    $this->table($headers, $users);

#### 進度條

對於長時間執行的任務，顯示進度條會很有幫助。使用輸出物件，我們可以開始、前進和停止進度條。先定義整個過程中過經過的步驟數量，然後處理完每個項目後再推進進度條：

    $users = App\User::all();

    $bar = $this->output->createProgressBar(count($users));

    $bar->start();

    foreach ($users as $user) {
        $this->performTask($user);

        $bar->advance();
    }

    $bar->finish();

更多進階選項，請查閱 [Symfony 進度條元件文件](https://symfony.com/doc/current/components/console/helpers/progressbar.html)

<a name="registering-commands"></a>
## 註冊指令

因為 `load` 方法呼叫了你在終端核心的 `commands` 方法，所有在 `app/Console/Commands` 資料夾中的指令都會自動註冊到 Artisan。事實上，你可以自由地呼叫 `load` 方法來掃描其他資料夾的 Artisan 指令：

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

你還可以藉由把指令的名稱寫入 `app/Console/Kernel.php` 資料夾中的 `$commands` 屬性來手動註冊指令。當 Artisan 啟動時，所有被列在這個屬性中的指令會被[服務容器](/docs/{{version}}/container) 解析並註冊到 Artisan 上：

    protected $commands = [
        Commands\SendEmails::class
    ];

<a name="programmatically-executing-commands"></a>
## 使用程式碼執行指令

有時候你會想要在指令列介面以外的地方執行 Artisan 指令。比如說，你希望在路由或控制器觸發 Artisan 指令，你可以使用`Artisan` facade 的 `call` 方法來達成。`call` 方法接受指令的名稱或類別做為第一個參數，指令參數的陣列作為第二個參數，並回傳退出碼：

    Route::get('/foo', function () {
        $exitCode = Artisan::call('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    });

或者，你也可以將整個 Artisan 指令的字串傳入 `call` 方法：

    Artisan::call('email:send 1 --queue=default');

`Artisan` facade 的 `queue` 方法可以將 Artisan 指令放進隊列裡，讓它由你的 [隊列進程](/docs/{{version}}/queues) 進行背景處理。在使用這個方法之前，請確保你已經設定好你的隊列和隊列監聽器：

    Route::get('/foo', function () {
        Artisan::queue('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    });

你也可以指定要將 Artisan 指令分配到哪個連接或隊列：

    Artisan::queue('email:send', [
        'user' => 1, '--queue' => 'default'
    ])->onConnection('redis')->onQueue('commands');

#### 傳送陣列

如果定義了接受陣列的指令，你可以將陣列值傳送給選項：

    Route::get('/foo', function () {
        $exitCode = Artisan::call('email:send', [
            'user' => 1, '--id' => [5, 13]
        ]);
    });

#### 傳送布林值

如果你需要指定非字串選項的值，例如 `migrate:refresh` 指令的 `--force` 標記，可以傳送 `true` 或 `false`：

    $exitCode = Artisan::call('migrate:refresh', [
        '--force' => true,
    ]);

<a name="calling-commands-from-other-commands"></a>
### 在指令中呼叫其他指令

有時候你希望在已經存在的 Artisan 指令中呼叫其他指令，你可以使用 `call` 方法。`call` 方法接受指令名稱和指令參數的陣列：

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

如果你想要呼叫另一個終端指令且抑制所有輸出，你可以使用 `callSilent` 方法。`callSilent` 方法和 `call` 方法有一樣的使用方式：

    $this->callSilent('email:send', [
        'user' => 1, '--queue' => 'default'
    ]);