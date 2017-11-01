# 資料庫：入們

- [介紹](#introduction)
    - [設定](#configuration)
    - [連線的寫入與讀取](#read-and-write-connections)
    - [使用多個資料庫的連線](#using-multiple-database-connections)
- [執行原始 SQL 查詢](#running-queries)
    - [監聽查詢事件](#listening-for-query-events)
- [資料庫事務](#database-transactions)

<a name="introduction"></a>
## 介紹

Laravel 使用原始 SQL 和[流暢的查詢產生器](/docs/{{version}}/queries)，以及還有 [Eloquent ORM](/docs/{{version}}/eloquent) 都可以非常簡單的操作各種資料庫後端系統。目前 Laravel 支援四種資料庫：

<div class="content-list" markdown="1">
- MySQL
- PostgreSQL
- SQLite
- SQL Server
</div>

<a name="configuration"></a>
### 設定

資料庫的設定檔放在 `config/database.php`。在這個設定檔內你可以定義所有的資料庫連接，以及指定預設使用哪個連接。在這個檔案內提供了大多數支援的資料庫系統範例。

預設來說，Laravel 的[環境設定](/docs/{{version}}/installation#environment-configuration)範例是使用 [Laravel Homestead](/docs/{{version}}/homestead)，在開發 Laravel 時，這是相當便利的本機虛擬機器。當然，你可以因應需求隨時修改你本機端的資料庫設定。

#### SQLite 設定

如果使用像是 `touch database/database.sqlite` 類似的指令建立新的 SQLite 資料庫，你可以使用資料庫的絕對路徑來輕易的將環境變數設定給剛建立的資料庫：

    DB_CONNECTION=sqlite
    DB_DATABASE=/absolute/path/to/database.sqlite

<a name="read-and-write-connections"></a>
### 分開讀與寫的連接

有時你可能希望為使用一個連線來處理查詢，其他的連線用來處理 寫入、更新和刪除。Laravel 使他變得輕而一舉，無論你使用原始查詢、查詢產生器或是 Eloquent ORM 都是可以使用的。

如何設定讀取與寫入的連接，讓我們看看這個例子：

    'mysql' => [
        'read' => [
            'host' => '192.168.1.1',
        ],
        'write' => [
            'host' => '196.168.1.2'
        ],
        'sticky'    => true,
        'driver'    => 'mysql',
        'database'  => 'database',
        'username'  => 'root',
        'password'  => '',
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix'    => '',
    ],

注意，有三個鍵加入了這個設定檔陣列內：`read`、`write` 和 `sticky`。`read` 和 `write` 鍵都包含單一個鍵的陣列值：`host`。`read` 及 `write` 連接的其他的資料庫設定選項將會合併在主要的 `mysql` 陣列內。

如果你希望覆蓋主要陣列的值，你只需要將項目放置在 `read` 和 `write` 的陣列中。所以，在這個情況下，`192.168.1.1` 會用於「讀取」的主機連線，而 `192.168.1.2` 會被用於「寫入」的主機練線。資料庫的憑證、前綴、編碼設定，以及所有其它的選項都存放在 `mysql` 陣列內，兩個連接將會共用這些選項。

#### `sticky` 選項

`sticky` 選項是可選擇的值，是被用來請求週期為終止前立即讀取剛寫入資料庫的資料。如果啟動了 `sticky` 選項，並且在請求週期終止前對資料庫進行「寫入」操作，那麼任何的「讀取」操作將會使用「寫入」的連線。這樣保證了在請求週期中寫入的任何資料可以在週期結束前立即從資料庫讀取剛寫入的資料。可以根據你的應用程式需求而決定是否要使用這個選項。

<a name="using-multiple-database-connections"></a>
### 使用多個資料庫的連線


當你使用多個連接，你可以使用 `DB` facade 的 `connection` 方法存取每個連線。傳遞給 `connection` 方法的 `name` 必須對應至 `config/database.php` 設定檔裡連接列表的其中一個：

    $users = DB::connection('foo')->select(...);

你也可以在連接的實例使用 `getPdo` 方法存取原始的底層 PDO 實例：

    $pdo = DB::connection()->getPdo();

<a name="running-queries"></a>
## 執行原始 SQL 查詢

一旦你設定好了資料庫連線，你可以使用 `DB` facade 進行查詢。`DB` facade 提供每個類型的查詢方法：`select`、`update`、`insert`、`delete`、`statement`。

#### 執行 Select 查詢

執行一個基本查詢，你可以在 `DB` facade 使用 `select`：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\DB;
    use App\Http\Controllers\Controller;

    class UserController extends Controller
    {
        /**
         * Show a list of all of the application's users.
         *
         * @return Response
         */
        public function index()
        {
            $users = DB::select('select * from users where active = ?', [1]);

            return view('user.index', ['users' => $users]);
        }
    }

傳遞給 `select` 方法的第一個參數是原始 SQL 查詢，而第二個參數是任何查詢需要的參數綁定。通常，這些是 `where` 子句的限定值。參數綁供了保護，為防止 SQL 注入。

`select` 方法總是回傳結果的`陣列`。陣列中的每個結果將是一個 PHP `StdClass` 物件，讓你能夠存取結果的值：

    foreach ($users as $user) {
        echo $user->name;
    }

#### 使用命名綁定

除了使用 `?` 表示你的參數綁定外，你也可以使用命名綁定執行查詢：

    $results = DB::select('select * from users where id = :id', ['id' => 1]);

#### 執行 Insert

若要執行 `insert` 語法，你可以在 `DB` facade 使用 `insert` 方法。如同 `select`，這個方法第一個參數是使用原始 SQL 查詢，第二個參數則是綁定：

    DB::insert('insert into users (id, name) values (?, ?)', [1, 'Dayle']);

#### 執行 Update

`update` 方法用於更新已經存在於資料庫的記錄。將會回傳該語法影響的行數：

    $affected = DB::update('update users set votes = 100 where name = ?', ['John']);

#### 執行 Delete

`delete` 方法用於刪除已經存在於資料庫的記錄。如同 `update`，受影響的行數將會被回傳：

    $deleted = DB::delete('delete from users');

#### 執行一般語法

某些資料庫語句不應該回傳任何值。對於這種類型的操作，你可以在 `DB` facade 使用 `statement` 方法：


    DB::statement('drop table users');

<a name="listening-for-query-events"></a>
### 監聽查詢事件

如果你希望能夠收到來自於你的應用程式每一筆 SQL 查詢，你可以使用 `listen` 方法。這個方法對於紀錄查詢跟除錯非常有用。你可以在[服務容器](/docs/{{version}}/providers)註冊你的查詢監聽器：

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\DB;
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
            DB::listen(function ($query) {
                // $query->sql
                // $query->bindings
                // $query->time
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

<a name="database-transactions"></a>
## 資料庫交易

你可以在 `DB` facade 上使用 `transaction` 方法來執行資料庫交易中的一組操作。如果在交易的`閉包`內拋出例外，交易會自動的被還原。如果`閉包`執行成功，交易將自動提交。你不需要擔心在使用 `transaction` 方法時手動還原或提交：

    DB::transaction(function () {
        DB::table('users')->update(['votes' => 1]);

        DB::table('posts')->delete();
    });

#### 處理死結

`transaction` 方法接受一個可選的第二個參數，它定義了當發生死結時交易應該重新嘗試的次數。一旦嘗試的次數被用完，將會拋出異常：

    DB::transaction(function () {
        DB::table('users')->update(['votes' => 1]);

        DB::table('posts')->delete();
    }, 5);

#### 手動操作交易

如果你想手動處理交易並且完全控制還原或提交，你可以在 `DB` facade 使用 `beginTransaction`：

    DB::beginTransaction();

你可以還原交易，透過 `rollBack` 方法：

    DB::rollBack();

最後，你可以提交這個交易，透過 `commit` 方法：

    DB::commit();

> {tip} `DB` facade 的交易方法控制[查詢建構器](/docs/{{version}}/queries)和 [Eloquent ORM](/docs/{{version}}/eloquent) 的交易。
