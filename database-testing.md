# 資料庫測試

- [介紹](#introduction)
- [產生工廠](#generating-factories)
- [每次測試完就重置資料庫](#resetting-the-database-after-each-test)
- [寫入工廠](#writing-factories)
    - [工廠狀態](#factory-states)
- [使用工廠](#using-factories)
    - [建立模型](#creating-models)
    - [保存模型](#persisting-models)
    - [關聯](#relationships)
- [可用的斷言](#available-assertions)

<a name="introduction"></a>
## 介紹

Laravel 提供了各種有用的工具，可以輕鬆的測試資料庫驅動。首先，你可以使用 `assertDatabaseHas` 輔助函式來判資料庫是否存在與指定條件相互匹配的資料。例如，如果想要驗證在 `user` 資料表中是否存在 `sally@example.com` 的 `email` 值，你可以執行以下操作來測試：

    public function testDatabase()
    {
        // Make call to application...

        $this->assertDatabaseHas('users', [
            'email' => 'sally@example.com'
        ]);
    }

你還可以使用 `assertDatabaseMissing` 輔助函式來判斷資料是否不存在於資料庫。

當然，`assertDatabaseHas` 方法和其他輔助函式一樣方便。你可以自由的使用任何 PHPUnit 的內建斷言方法來完善你的測試。

<a name="generating-factories"></a>
## 產生工廠

使用 [Artisan 指令](/docs/{{version}}/artisan) 的 `make:factory` 來產生工廠：

    php artisan make:factory PostFactory

新的工廠會放在你的 `database/factories` 目錄。

`--model` 選項可用於指示由工廠建立的模型名稱。這個選項會使用給定的模型來預先填充產生的工廠檔案。

    php artisan make:factory PostFactory --model=Post

<a name="resetting-the-database-after-each-test"></a>
## 每次測試完就重置資料庫

在每次測試完就重置資料庫，這樣有助於之前測試用的資料不會影響下一次測試。`RefreshDatabase` trait 是根據你使用記憶體資料庫或傳統資料庫來遷移你的測試資料庫的最佳方式。只需使用 trait 在你的測試類別上，並為你處理所有事情：

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;
    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;

    class ExampleTest extends TestCase
    {
        use RefreshDatabase;

        /**
         * A basic functional test example.
         *
         * @return void
         */
        public function testBasicExample()
        {
            $response = $this->get('/');

            // ...
        }
    }

<a name="writing-factories"></a>
## 寫入工廠

在測試時，你可能需要在測試前對資料庫寫入幾筆記錄。建立這個測試資料的時候不用手動指定每列的值，因為 Laravel 可以讓你使用模型工廠來為每個 [Eloquent 模型](/docs/{{version}}/eloquent)定義一組預設的屬性。請查看 `database/factories/UserFactory.php` 檔案，這個檔案已有現成的工廠定義：

    use Faker\Generator as Faker;

    $factory->define(App\User::class, function (Faker $faker) {
        static $password;

        return [
            'name' => $faker->name,
            'email' => $faker->unique()->safeEmail,
            'password' => $password ?: $password = bcrypt('secret'),
            'remember_token' => str_random(10),
        ];
    });

在閉包中的是對工廠的定義，你可以回傳模型上所有屬性的預設測試值。閉包會接收一個 [Faker](https://github.com/fzaninotto/Faker) PHP 程式庫的實例，這會讓你方便的產生各種隨機資料來測試。

你也可以為了更好組織每個模型而去建立額外的工廠檔案。例如，你應該建立 `UserFactory.php` 和 `CommentFactory.php` 檔案在你的 `database/factories` 目錄中。工廠目錄中的所有檔案會自動被 Laravel 載入。

<a name="factory-states"></a>
### 工廠狀態

工廠狀態可以定義隨機修改狀態，並且能被應用在任何組合上的模型工廠。例如，你的 `User` 模型或許有修改過其中一項預設的屬性值，所以會用 `delinquent` 狀態。你可以使用 `state` 方法來定義你的狀態變化。你可以傳入要修改的屬性陣列來簡單的改變狀態：

    $factory->state(App\User::class, 'delinquent', [
        'account_status' => 'delinquent',
    ]);

如果你的狀態需要計算或 `$faker` 實例，你可以使用閉包來計算狀態的屬性修改：

    $factory->state(App\User::class, 'address', function ($faker) {
        return [
            'address' => $faker->address,
        ];
    });

<a name="using-factories"></a>
## 使用工廠

<a name="creating-models"></a>
### 建立模型

一旦你定義了工廠，你可以在你的測試或種子檔案使用全域的 `factory` 函式來產生模型實例。接著，讓我們看看看幾個創建模型的範例。首先，我們會使用 `make` 來建立模型，但不是儲存它們到資料庫：

    public function testDatabase()
    {
        $user = factory(App\User::class)->make();

        // Use model in tests...
    }

你也可以建立多個模型集合或建立給定類型的模型：

    // Create three App\User instances...
    $users = factory(App\User::class, 3)->make();

#### 應用工廠狀態

你也可以應用你的任何[狀態](#factory-states)給模型。如果你想要模型的應用有多種狀態變化，你應該指定要應用的每個狀態名稱：

    $users = factory(App\User::class, 5)->states('delinquent')->make();

    $users = factory(App\User::class, 5)->states('premium', 'delinquent')->make();

#### 覆蓋屬性

如果你想要覆蓋模型中的某些預設值，你可以將一組陣列值傳到 `make` 方法。只有指定的值會被替換，而剩下的值將維持工廠指定的預設值來設定：

    $user = factory(App\User::class)->make([
        'name' => 'Abigail',
    ]);

<a name="persisting-models"></a>
### 保存模型

`create` 方法不只建立模型實例，還會使用 Eloquent 的 `save` 方法來儲存它們到資料庫：

    public function testDatabase()
    {
        // Create a single App\User instance...
        $user = factory(App\User::class)->create();

        // Create three App\User instances...
        $users = factory(App\User::class, 3)->create();

        // Use model in tests...
    }

你可以在模型上傳遞一組陣列到 `create` 方法來覆蓋屬性：

    $user = factory(App\User::class)->create([
        'name' => 'Abigail',
    ]);

<a name="relationships"></a>
### 關聯

在這個範例中，我們會嘗試關聯某些已建好的模型。當使用 `create` 方法來建立多型模型時，會回傳 Eloquent 的 [集合實例](/docs/{{version}}/eloquent-collections)，可以讓你使用集合提供的任何方便的功能，像是 `each`：

    $users = factory(App\User::class, 3)
               ->create()
               ->each(function ($u) {
                    $u->posts()->save(factory(App\Post::class)->make());
                });

#### 關聯與屬性閉包

你也可以嘗試在工廠定義中使用閉包屬性來關聯模型。例如，如果你想要在建立 `Post` 模型的時候建立新的 `User` 實例，你可以效仿下述：

    $factory->define(App\Post::class, function ($faker) {
        return [
            'title' => $faker->title,
            'content' => $faker->paragraph,
            'user_id' => function () {
                return factory(App\User::class)->create()->id;
            }
        ];
    });

這些閉包也接收定義它們的工廠陣列所要的評估屬性：

    $factory->define(App\Post::class, function ($faker) {
        return [
            'title' => $faker->title,
            'content' => $faker->paragraph,
            'user_id' => function () {
                return factory(App\User::class)->create()->id;
            },
            'user_type' => function (array $post) {
                return App\User::find($post['user_id'])->type;
            }
        ];
    });

<a name="available-assertions"></a>
## 可用的斷言

Laravel 為你 [PHPUnit](https://phpunit.de/) 測試提供了幾個資料庫斷言：

Method  | 說明
------------- | -------------
`$this->assertDatabaseHas($table, array $data);`  |  Assert that a table in the database contains the given data.
`$this->assertDatabaseMissing($table, array $data);`  |  Assert that a table in the database does not contain the given data.
`$this->assertSoftDeleted($table, array $data);`  |  Assert that the given record has been soft deleted.
