# 資料庫：遷移

- [介紹](#introduction)
- [產生遷移](#generating-migrations)
- [遷移結構](#migration-structure)
- [執行遷移](#running-migrations)
    - [還原遷移](#rolling-back-migrations)
- [資料表](#tables)
    - [建立資料表](#creating-tables)
    - [重新命名與移除資料表](#renaming-and-dropping-tables)
- [欄位](#columns)
    - [建立欄位](#creating-columns)
    - [欄位修飾](#column-modifiers)
    - [修改欄位](#modifying-columns)
    - [移除欄位](#dropping-columns)
- [索引](#indexes)
    - [建立索引](#creating-indexes)
    - [移除索引](#dropping-indexes)
    - [外鍵約束](#foreign-key-constraints)

<a name="introduction"></a>
## 介紹

遷移，可以說是資料庫的版本控制，可以讓你的團隊輕易修改與共享應用程式的資料庫結構。遷移通常會搭配 Laravel 的 schema 建構器來輕易的建構資料庫的結構。如果你曾有過不得不告知團隊要手動新增欄位到各自的本機資料庫結構的情況，那你可以使用資料庫遷移來解決這個問題。

Laravel `Schema` [facade](/docs/{{version}}/facades) 為所有 Laravel 支援的資料庫系統提供建立和操作資料庫的相容性支援。

<a name="generating-migrations"></a>
## 產生遷移

使用 [Artisan 指令](/docs/{{version}}/artisan) 的 `make:migration` 來建立遷移：

    php artisan make:migration create_users_table

新遷移會放置於 `database/migrations` 目錄中。每個遷移的檔名會包含時間戳，可以 Laravel 確定遷移的順序。

`--table` 和 `--create` 選項也可以被用於指定資料表名稱和是否依遷移內容建立新資料表。這些選項只在剛產生遷移時填入指定的資料表：

    php artisan make:migration create_users_table --create=users

    php artisan make:migration add_votes_to_users_table --table=users

如果你希望為產生的遷移指定自訂的輸出路徑。你可以在執行時 `make:migration` 指令的時候使用 `--path` 選項。給定的路徑應該要是應用程式基本路徑的相對路徑。

<a name="migration-structure"></a>
## 遷移結構

遷移類別會有兩個方法：`up` 和 `down`。 `up` 方法被用於新增新資料表，欄位，或索引到你的資料庫，而 `down` 方法則會回朔 `up` 的執行操作。

你可以在這兩種方法中使用 Laravel schema 建構器來建立和修改的資料表。想學習 `Schema` 建構器可用的所有方法，[請查看該文件](#creating-tables)。例如，這個遷移範例會建立 `flights` 資料表：

    <?php

    use Illuminate\Support\Facades\Schema;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Database\Migrations\Migration;

    class CreateFlightsTable extends Migration
    {
        /**
         * 執行遷移。
         *
         * @return void
         */
        public function up()
        {
            Schema::create('flights', function (Blueprint $table) {
                $table->increments('id');
                $table->string('name');
                $table->string('airline');
                $table->timestamps();
            });
        }

        /**
         * 回朔遷移。
         *
         * @return void
         */
        public function down()
        {
            Schema::drop('flights');
        }
    }

<a name="running-migrations"></a>
## 執行遷移

你可以執行 Artisan 指令的 `migrate` 來執行所有未完成的遷移：

    php artisan migrate

> {note} 如果你使用 [Homestead 虛擬機](/docs/{{version}}/homestead)，你應該從你的虛擬機中執行這個指令。

#### 在正式上線環境強制執行遷移

有些遷移操作是不可逆的，也就意味著這些操作可能會導致你遺失資料。為了預防你在正式上線的資料庫上執行這些指令，系統會在執行指令前詢問你是否要執行該指令。想不用被提示的情況下強制執行指令，可使用 `--force` 選項：

    php artisan migrate --force

<a name="rolling-back-migrations"></a>
### 還原遷移

你可以使用 `rollback` 指令來還原最後的遷移操作。這個指令還原最後批次的遷移，這可能包含多個遷移檔案：

    php artisan migrate:rollback

你可以提供在 `rollback` 指令加上 `step` 選項來還原限定次數的遷移。例如，以下的指令會還原最後五次遷移：

    php artisan migrate:rollback --step=5

`migrate:reset` 指令會還原所有應用程式的遷移：

    php artisan migrate:reset

#### 使用單一指令來還原與遷移

`migrate:refresh` 指令會還原所有遷移，然後執行 `migrate` 指令。這個指令可有效的重建整個資料庫：

    php artisan migrate:refresh

    // 覆寫資料庫並執行資料庫填充...
    php artisan migrate:refresh --seed

你可以在 `refresh` 中提供 `step` 選項來指定次數的還原與重新遷移。例如，以下指令會還原與重新遷移最後五次的遷移：

    php artisan migrate:refresh --step=5

#### 移除所有資料資料表與遷移

`migrate:fresh` 指令會移除資料庫的所有資料表，然後執行 `migrate` 指令：

    php artisan migrate:fresh

    php artisan migrate:fresh --seed

<a name="tables"></a>
## 資料表

<a name="creating-tables"></a>
### 建立資料表

在 `Schema` facade 上使用 `create` 方法來建立新的資料表。`create` 方法可接受兩個參數：其一是資料表名稱，其二是`閉包`。閉包可接收被用於定義新資料表的 `Blueprint` 物件：

    Schema::create('users', function (Blueprint $table) {
        $table->increments('id');
    });

當然，你可以在建立資料表的時候，使用任何 schema 建構器的[欄位方法](#creating-columns)來定義資料表的欄位。

#### 檢查資料表與欄位是否存在

你可以使用 `hasTable` 和 `hasColumn` 方法來輕易地檢查資料表或欄位是否存在：

    if (Schema::hasTable('users')) {
        //
    }

    if (Schema::hasColumn('users', 'email')) {
        //
    }

#### 連線與儲存的引擎

如果你想對非預設的資料庫連線執行 schema 操作，可使用 `connection` 方法：

    Schema::connection('foo')->create('users', function (Blueprint $table) {
        $table->increments('id');
    });

你可以在 schema 建構器上使用 `engine` 屬性來定義資料表的儲存引擎：

    Schema::create('users', function (Blueprint $table) {
        $table->engine = 'InnoDB';

        $table->increments('id');
    });

<a name="renaming-and-dropping-tables"></a>
### 重新命名與移除資料表

你可以使用 `rename` 方法來重新命名已存在的資料表：

    Schema::rename($from, $to);

你可以使用 `drop` 或 `dropIfExists` 方法來移除已存在的資料表：

    Schema::drop('users');

    Schema::dropIfExists('users');

#### 重新命名已有外鍵的資料表

在重新命名資料表前，你應該驗證在資料表上的任何外鍵約束在遷移檔案中都有明確的名稱，而不是讓 Laravel 發給你的基本約束名稱。否則外鍵約束名稱會沿用舊資料表。

<a name="columns"></a>
## 欄位

<a name="creating-columns"></a>
### 建立欄位

在 `Schema` facade 上的 `table` 方法可被用於更新已存在的資料表。像是 `create` 方法、`table` 方法可接受兩個參數：資料表名稱和`閉包`，然後閉包可接受一個 `Blueprint` 實例，可用於新增欄位到資料表：

    Schema::table('users', function (Blueprint $table) {
        $table->string('email');
    });

#### 可用的欄位型別

當然，schema 建構器包含各種欄位類型，你可以在建構資料表的時候指定這些方法：

指令  | 說明
------------- | -------------
`$table->bigIncrements('id');`  |  自動遞增相當於 UNSIGNED BIGINT（主鍵）欄位。
`$table->bigInteger('votes');`  |  相當於 BIGINT 欄位。
`$table->binary('data');`  |  相當於 BLOB 欄位。
`$table->boolean('confirmed');`  |  相當於 BOOLEAN 欄位
`$table->char('name', 100);`  |  相當於 CHAR 欄位並可選擇長度。
`$table->date('created_at');`  |  相當於 DATE 欄位。
`$table->dateTime('created_at');`  |  相當於 DATETIME 對等欄位。
`$table->dateTimeTz('created_at');`  |  相當於 DATETIME （含時區） 欄位。
`$table->decimal('amount', 8, 2);`  |  相當於 DECIMAL 欄位，並帶有精度（整數位長度）與基數（小數點長度）。
`$table->double('amount', 8, 2);`  |  相當於 DOUBLE 欄位，並帶有精度（整數位長度）與基數（小數點長度）。
`$table->enum('level', ['easy', 'hard']);`  |  相當於 ENUM 欄位。
`$table->float('amount', 8, 2);`  |  相當於 FLOAT 欄位，並帶有精度（整數位長度）與基數（小數點長度）。
`$table->geometry('positions');`  |  相當於 GEOMETRY 欄位。
`$table->geometryCollection('positions');`  |  相當於 GEOMETRYCOLLECTION 欄位。
`$table->increments('id');`  |  自動遞增相當於 UNSIGNED INTEGER（主鍵）欄位。
`$table->integer('votes');`  |  相當於 INTEGER 欄位。
`$table->ipAddress('visitor');`  |  相當於 IP 位址欄位。
`$table->json('options');`  |  相當於 JSON 欄位。
`$table->jsonb('options');`  |  相當於 JSONB 欄位。
`$table->lineString('positions');`  |  相當於 LINESTRING 欄位。
`$table->longText('description');`  |  相當於 LONGTEXT 欄位。
`$table->macAddress('device');`  |  相當於 MAC 位址欄位。
`$table->mediumIncrements('id');`  |  自動遞增相當於 UNSIGNED MEDIUMINT（主鍵）欄位。
`$table->mediumInteger('votes');`  |  相當於 MEDIUMINT 欄位。
`$table->mediumText('description');`  |  相當於 MEDIUMTEXT 欄位。
`$table->morphs('taggable');`  |  新增相當於 `taggable_id` 的 UNSIGNED INTEGER 和 `taggable_type` 的 VARCHAR 欄位。
`$table->multiLineString('positions');`  |  相當於 MULTILINESTRING 欄位。
`$table->multiPoint('positions');`  |  相當於 MULTIPOINT 欄位。
`$table->multiPolygon('positions');`  |  相當於 MULTIPOLYGON 欄位。
`$table->nullableMorphs('taggable');`  |  新增可以 null 的 `morphs()` 欄位版本。
`$table->nullableTimestamps();`  |  新增可以 null 的 `timestamps()` 欄位版本。
`$table->point('position');`  |  相當於 POINT 欄位。
`$table->polygon('positions');`  |  相當於 POLYGON 欄位。
`$table->rememberToken();`  |  相當於新增可以 null 的 `remember_token` VARCHAR(100) 欄位。
`$table->smallIncrements('id');`  |  自動遞增相當於 UNSIGNED SMALLINT（主鍵）欄位。
`$table->smallInteger('votes');`  |  相當於 SMALLINT 欄位。
`$table->softDeletes();`  |  相當於為緩刪除新增可以 null 的 `deleted_at` 時間戳欄位。
`$table->softDeletesTz();`  |  相當於為緩刪除新增可以 null 的 `deleted_at` 時間戳（含時區）欄位。
`$table->string('name', 100);`  |  相當於 VARCHAR 欄位並可選擇長度。
`$table->text('description');`  |  相當於 TEXT 欄位。
`$table->time('sunrise');`  |  相當於 TIME 欄位。
`$table->timeTz('sunrise');`  |  相當於 TIME（含時區）欄位。
`$table->timestamp('added_on');`  |  相當於 TIMESTAMP 欄位。
`$table->timestampTz('added_on');`  |  相當於 TIMESTAMP（含時區）欄位。
`$table->timestamps();`  |  相當於新增可以 null 的 `created_at` 和 `updated_at` 時間戳欄位。
`$table->timestampsTz();`  |  相當於新增可以 null 的 `created_at` 和 `updated_at` 時間戳（含時區）欄位。
`$table->tinyIncrements('id');`  |  自動遞增相當於 UNSIGNED TINYINT（主鍵）欄位。
`$table->tinyInteger('votes');`  |  相當於 TINYINT 欄位欄位。
`$table->unsignedBigInteger('votes');`  |  相當於 UNSIGNED BIGINT 欄位。
`$table->unsignedDecimal('amount', 8, 2);`  |  相當於 UNSIGNED DECIMAL 欄位，並帶有精度（整數位長度）與基數（小數點長度）。
`$table->unsignedInteger('votes');`  |  相當於 UNSIGNED INTEGER 欄位。
`$table->unsignedMediumInteger('votes');`  |  相當於 UNSIGNED MEDIUMINT 欄位。
`$table->unsignedSmallInteger('votes');`  |  相當於 UNSIGNED SMALLINT 欄位。
`$table->unsignedTinyInteger('votes');`  |  相當於 UNSIGNED TINYINT 欄位。
`$table->uuid('id');`  |  相當於 UUID 欄位。

<a name="column-modifiers"></a>
### 欄位修飾

除了以上的欄位類型清單，這有幾個欄位「修飾」可以用於新增欄位到資料表。例如，你可以使用 `nullable` 方法來使欄位「nullable」：

    Schema::table('users', function (Blueprint $table) {
        $table->string('email')->nullable();
    });

以下是所有可用的欄位修飾清單，這個清單不含 [索引修飾](#creating-indexes)：

修飾方法  | 說明
------------- | -------------
`->after('column')`  |  將該欄位放置其他欄位「之後」。（MySQL 限定）
`->autoIncrement()`  |  設定 INTEGER 欄位能自動遞增（主鍵）
`->charset('utf8')`  |  為欄位指定字元編碼。（MySQL 限定）
`->collation('utf8_unicode_ci')`  |  為欄位指定序。（MySQL 與 SQL 伺服器限定）
`->comment('my comment')`  |  為欄位新增註解。 （MySQL 限定）
`->default($value)`  |  為欄位指定「預設」值。
`->first()`  |  將此欄位放置於資料表的「第一個」。（MySQL 限定）
`->nullable($value = true)`  |  讓欄位預設允許寫入 NULL 值。
`->storedAs($expression)`  |  建立一個儲存產生的欄位。（MySQL 限定）
`->unsigned()`  |  設定 INTEGER 欄位為 UNSIGNED。 （MySQL 限定）
`->useCurrent()`  |  設定 TIMESTAMP 欄位要使用 CURRENT_TIMESTAMP 作為預設值。
`->virtualAs($expression)`  |  建立一個虛擬產生的欄位。（MySQL 限定）

<a name="modifying-columns"></a>
### 修改欄位

#### 必要條件

在修改欄位之前，請務必在你的 `composer.json` 檔案新增 `doctrine/dbal` 依賴。Doctrine DBAL 函式庫被用於判斷當前欄位狀態與建立調整指定欄位的 SQL 查詢：

    composer require doctrine/dbal

#### 更新欄位屬性

`change` 方法可以讓你修改一些已存在的欄位類型或修改欄位屬性。例如，你可能希望增加字串欄位的長度。要查看 `change` 方法的作用，讓我們將 `name` 欄位的大小從 25 改成 50：

    Schema::table('users', function (Blueprint $table) {
        $table->string('name', 50)->change();
    });

我們也能將欄位修改為 nullable：

    Schema::table('users', function (Blueprint $table) {
        $table->string('name', 50)->nullable()->change();
    });

> {note} 以下欄位類型不能被「更改」：char、double、enum、mediumInteger、timestamp、tinyInteger、ipAddress、json、jsonb、macAddress、mediumIncrements、morphs、nullableMorphs、nullableTimestamps、softDeletes、timeTz、timestampTz、timestamps、timestampsTz、unsignedMediumInteger、unsignedTinyInteger、uuid。

#### 重新命名欄位

你可以在 Schema 建構器上使用 `renameColumn` 方法來重新命名欄位。再重新命名欄位之前，請務必在你的 `composer.json` 檔案上新增 `doctrine/dbal` 依賴：

    Schema::table('users', function (Blueprint $table) {
        $table->renameColumn('from', 'to');
    });

> {note} 目前不支援修改 `enum` 類型的欄位名稱。

<a name="dropping-columns"></a>
### 移除欄位

你可以在 Schema 建構器上使用 `dropColumn` 方法來移除欄位。從 SQLite 資料庫移除欄位之前，你會需要在你的 `composer.json` 檔案上新增 `doctrine/dbal` 依賴，並在你的終端機執行 `composer update`來安裝該函式庫：

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('votes');
    });

你可以從資料表的上傳入欄位名稱陣列到 `dropColumn` 方法來移除多個欄位：

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn(['votes', 'avatar', 'location']);
    });

> {note} 當使用 SQLite 資料庫時，並不支援在單一遷移中移除或修改多個欄位。

<a name="indexes"></a>
## 索引

<a name="creating-indexes"></a>
### 建立索引

schema 建構器支援幾個索引類型。首先，讓我們看一下指定欄位的值必須是 unique 的範例。要建立索引，我們可以只鏈結 `unique` 方法到欄位定義上：

    $table->string('email')->unique();

或者，你可以定義欄位後才建立索引。例如：

    $table->unique('email');

你甚至可以將欄位的陣列傳入 `index` 方法來建立複合索引：

    $table->index(['account_id', 'created_at']);

Laravel 會自動產生一個合理的索引名稱，但你可以指定你要的名稱作為第二參數，並傳入該方法：

    $table->unique('email', 'unique_email');

#### 可用的索引類型

指令  | 說明
------------- | -------------
`$table->primary('id');`  |  新增主鍵。
`$table->primary(['id', 'parent_id']);`  |  新增複合鍵。
`$table->unique('email');`  |  新增唯一的索引。
`$table->index('state');`  |  新增基本的索引。
`$table->spatialIndex('location');`  |  新增空間索引。（MySQL 限定）

#### 索引長度與 MySQL 與 MariaDB

Laravel 預設使用 `utf8mb4` 字元編碼， 包括支援在資料庫中儲存「表情符號」。如果你執行的 MySQL 版本比 5.5.7 版本還舊，或 MariaDB 還沒升級至 10.2.2 以上，又想要在遷移被產生時 MySQL 能順利的為它們建立索引，你需要手動設定預設字串長度。你可以在 `AppServiceProvider` 中呼叫 `Schema::defaultStringLength` 方法來設定：

    use Illuminate\Support\Facades\Schema;

    /**
     * 引導任何應用程式服務。
     *
     * @return void
     */
    public function boot()
    {
        Schema::defaultStringLength(191);
    }

或者，你可以為資料庫啟用 `innodb_large_prefix` 選項。關於如何啟用這個選項的說明，請參考你所使用的資料庫文件。

<a name="dropping-indexes"></a>
### 移除索引

要移除索引，你必須指定該索引的名稱。Laravel 預設會自動為索引分配一個合理的名稱。只要將資料表名稱、索引欄位名稱和索引類型連接再一起。這裡有一些範例：

指令  | 說明
------------- | -------------
`$table->dropPrimary('users_id_primary');`  |  從「users」資料表移除主鍵。
`$table->dropUnique('users_email_unique');`  |  從「users」資料表移除唯一索引。
`$table->dropIndex('geo_state_index');`  |  從「geo」資料表移除基本的索引。

如果你傳入欄位陣列到 `dropIndex` 方法，會根據資料表名稱、欄位和鍵的類型來產生要移除的預設索引名稱：

    Schema::table('geo', function (Blueprint $table) {
        $table->dropIndex(['state']); // 移除 'geo_state_index' 索引
    });

<a name="foreign-key-constraints"></a>
### 外鍵約束

Laravel 也為建立外鍵約束提供支援，這是用於在資料庫層面強制參考完整性。例如，讓我們定義 posts 資料表內有個 user_id 欄位要參考至 users 資料表的 id 欄位：

    Schema::table('posts', function (Blueprint $table) {
        $table->integer('user_id')->unsigned();

        $table->foreign('user_id')->references('id')->on('users');
    });

你也可以指定約束的「on delete」及「on update」作為所需的操作：

    $table->foreign('user_id')
          ->references('id')->on('users')
          ->onDelete('cascade');

要移除外鍵，你可以使用 dropForeign 方法。外鍵約束與索引採用相同的命名方式。所以，我們可以連結資料表明與約束的欄位，接著在該名稱加上「_foreign」後綴：

    $table->dropForeign('posts_user_id_foreign');

或者你可以傳入陣列值，在移除時會自動使用預設約束名稱：

    $table->dropForeign(['user_id']);

你可以在你的遷移中使用下列方法來啟用或停用外鍵約束：

    Schema::enableForeignKeyConstraints();

    Schema::disableForeignKeyConstraints();
