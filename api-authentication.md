# API Authentication

- [介紹](#introduction)
- [設定](#configuration)
    - [資料庫準備](#database-preparation)
- [產生 Tokens](#generating-tokens)
    - [Hashing Tokens](#hashing-tokens)
- [路由保護](#protecting-routes)
- [請求中傳送 Tokens](#passing-tokens-in-requests)

<a name="introduction"></a>
## 介紹

預設情況下，Laravel 為 API 認證提供一種簡單的解決方案，透過分配一個隨機 token 給你的應用程式中的使用者。在你的 `config/auth.php` 設定檔中，已經定義了一個 `api` guard 並且使用 `token` 驅動。這個驅動負責檢查傳入請求的 API token 並驗證它是否符合資料庫中分配給使用者的 Token。

> **注意：** 儘管 Laravel 提供了簡單的、基於 token 的驗證保護，我們仍強烈建議你考慮以 [Laravel Passport](/docs/{{version}}/passport) 來實現一個提供 API 認證的健全的、可用於生產的應用程式。

<a name="configuration"></a>
## 設定

<a name="database-preparation"></a>
### 資料庫準備

在使用 `token` 驅動之前，你需要先[創建一個遷移檔](/docs/{{version}}/migrations)，這個遷移檔在你的 `users` 資料表中新增一個 `api_token` 欄位。

    Schema::table('users', function ($table) {
        $table->string('api_token', 80)->after('password')
                            ->unique()
                            ->nullable()
                            ->default(null);
    });

創建遷移檔之後，執行 `migrate` Artisan 指令。

> {tip} 如果你選擇使用不同的欄位名稱，請確保在 `config/auth.php` 設定檔中更新 API 的 `storage_key` 設定選項。

<a name="generating-tokens"></a>
## 產生 Tokens

將 `api_token` 欄位新增到你的 `users` 資料表後，你就可以將隨機的 API tokens 分配給每一個註冊你的應用程式的使用者了。你應該在註冊期間 `User` 模型被建立時分配這些 tokens。當使用 `laravel/ui` Composer 套件提供的[認證框架](/docs/{{version}}/authentication#authentication-quickstart)時，可以在 `RegisterController` 中的 `create` 方法中完成此操作：

    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Str;

    /**
     * Create a new user instance after a valid registration.
     *
     * @param  array  $data
     * @return \App\User
     */
    protected function create(array $data)
    {
        return User::forceCreate([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
            'api_token' => Str::random(80),
        ]);
    }

<a name="hashing-tokens"></a>
### Hashing Tokens

在上面的範例中，API tokens 以純文字的方式被儲存在你的資料庫中。如果你想要用 SHA-256 雜湊演算法來 hash 你的 API tokens，可以將你的 `api` guard 設定中的 `hash` 選項設為 `true`。`api` guard 被定義在你的 `config/auth.php` 設定檔：

    'api' => [
        'driver' => 'token',
        'provider' => 'users',
        'hash' => true,
    ],

#### 產生 Hashed Tokens

當使用 hashed API tokens 時，你不應該在使用者註冊階段產生API tokens。作為替代，你需要在你的應用程式中實做自己的 API token 管理頁面。這個頁面應該要允許使用者初始化和刷新他們的 API token。當使用者發出初始化或刷新 token 的請求時，你應該要將一份 token 的 hash 副本存在資料庫中，並且返回純文字的 token 副本給 view / 前端客戶端進行一次性的顯示。

舉個例子，一個用於初始化/刷新給定用戶的 token 並以 JSON 回應的格式返回純文字 token 的控制器方法可能類似以下內容：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use Illuminate\Support\Str;

    class ApiTokenController extends Controller
    {
        /**
         * Update the authenticated user's API token.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return array
         */
        public function update(Request $request)
        {
            $token = Str::random(80);

            $request->user()->forceFill([
                'api_token' => hash('sha256', $token),
            ])->save();

            return ['token' => $token];
        }
    }

> {tip} 由於上面範例中的 API tokens 具有足夠的熵(entropy)，創建「彩虹表」查找 hashed token 原始值是不切實際的，所以並不需要如 `bcrypt` 的 slow hashing 方法。

<a name="protecting-routes"></a>
## 路由保護

Laravel 包含一個[認證保護](/docs/{{version}}/authentication#adding-custom-guards)，它將自動驗證傳入請求的 API tokens。你只需要在任何要求有效的訪問 token 的路由上指定 `auth:api` 中介層：

    use Illuminate\Http\Request;

    Route::middleware('auth:api')->get('/user', function (Request $request) {
        return $request->user();
    });

<a name="passing-tokens-in-requests"></a>
## 請求中傳送 Tokens

有幾個方法可以傳送 API token 到你的應用程式。我們將在使用 Guzzle HTTP 函式庫展示他們的用法時去討論這些方法，你可以根據應用程式的需求選擇其中任何一種。

#### 請求參數

你的應用程式的 API 使用者可以將其 token 指定為 `api_token` 查詢字串：

    $response = $client->request('GET', '/api/user?api_token='.$token);

#### 請求負載

你的應用程式的 API 使用者可以將其 API token 做為 `api_token` 包含在請求的表單參數中：

    $response = $client->request('POST', '/api/user', [
        'headers' => [
            'Accept' => 'application/json',
        ],
        'form_params' => [
            'api_token' => $token,
        ],
    ]);

#### Bearer Token

你的應用程式的 API 使用者可以在請求的 `Authorization` 標頭中提供其 API token 做為 `Bearer` token：

    $response = $client->request('POST', '/api/user', [
        'headers' => [
            'Authorization' => 'Bearer '.$token,
            'Accept' => 'application/json',
        ],
    ]);