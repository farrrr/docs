# 授權

- [介紹](#introduction)
- [Gates](#gates)
    - [撰寫 Gates](#writing-gates)
    - [授權行為](#authorizing-actions-via-gates)
- [建立原則](#creating-policies)
    - [產生原則](#generating-policies)
    - [註冊原則](#registering-policies)
- [撰寫原則](#writing-policies)
    - [原則方法](#policy-methods)
    - [無法核對授權資料的情況](#methods-without-models)
    - [原則篩選器](#policy-filters)
- [使用原則來授權行為](#authorizing-actions-using-policies)
    - [透過使用者模型](#via-the-user-model)
    - [透過中介層](#via-middleware)
    - [透過控制器的輔助函式](#via-controller-helpers)
    - [透過 Blade 模板](#via-blade-templates)

<a name="introduction"></a>
## 介紹

除了內建提供的[認證](/docs/{{version}}/authentication)服務外， Laravel也提供了簡單的方式來組織認證邏輯和控制資源的存取。像是認證一般， Laravel 授權功能是採用簡單的兩種主要方式來授權行為：「 Gates 」與「原則」

「 Gates 」和「原則」的關係像是「路由」和「控制器」。「 Gates 」提供簡單且採用閉包的方式去授權，而「原則」像是控制器，使用程式邏輯去區分它們特定的模型或資源。我們先來看看「 Gates 」，然後再看看「原則」。

在應用程式中，你不需要在 Gates 與 原則之間做選擇。大多應用程式極有可能同時使用 Gates 和原則，且這麼做還蠻推薦的！ Gates 大部分應用在模型和資源無關的地方，比如瀏覽後台儀表板。相反的，原則的應用必須使用到特定的模型與資源。

<a name="gates"></a>
## Gates

<a name="writing-gates"></a>
### 撰寫 Gates

Gates 是用來確認是否受權指定的行為給使用者，以及通常在 `App\Providers\AuthServiceProvider` 的類別中使用 facade `Gate` 定義內容。Gates 的第一個參數主要接受使用者實例，並且可以接受額外的參數，像是 Eloquent 模型關聯：

    /**
     * 註冊任何認證或授權的服務
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Gate::define('update-post', function ($user, $post) {
            return $user->id == $post->user_id;
        });
    }

Gates 也可以定義一個用 `Class@method` 風格的回呼字串，像是使用控制器那樣：

    /**
     * 註冊任何認證或授權的服務
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Gate::define('update-post', 'PostPolicy@update');
    }

#### Gates 的 resource 方法

你也可以使用 `resource`方法來一次定義多個 Gate 功能：

    Gate::resource('posts', 'PostPolicy');

這個定義與下面手動去刻的定義相同：

    Gate::define('posts.view', 'PostPolicy@view');
    Gate::define('posts.create', 'PostPolicy@create');
    Gate::define('posts.update', 'PostPolicy@update');
    Gate::define('posts.delete', 'PostPolicy@delete');

預設情況下，會定義 `view` 、 `create` 、 `update` 和 `delete` 這些功能。你可以傳遞陣列到第三個參數來擴充或覆蓋 `resource` 方法給的功能。陣列的鍵定義功能的名稱，而陣列的值定義方法的名稱。舉個例子，下列的程式碼將建立兩個新的 Gate 定義 —— `posts.image` 和 `posts.photo`：

    Gate::resource('posts', 'PostPolicy', [
        'image' => 'updateImage',
        'photo' => 'updatePhoto',
    ]);

<a name="authorizing-actions-via-gates"></a>
### 授權行為

使用 Gates 來授權行為，你應當使用 `allows` 或 `denies` 方法。請注意！你不需要再傳遞當前認證通過的使用者這些方法。 Laravel 會自己注意通過 Gate 閉包的使用者：

    if (Gate::allows('update-post', $post)) {
        // 如果這個使用者可以更新貼文就...
    }

    if (Gate::denies('update-post', $post)) {
        // 如果這個使用者不能更新貼文就...
    }

如果你需要確定是否有授權行為給使用者，你可以使用 facade `Gate` 的 `forUser` 方法：

    if (Gate::forUser($user)->allows('update-post', $post)) {
        // 如果這個使用者可以更新貼文就...
    }

    if (Gate::forUser($user)->denies('update-post', $post)) {
        // 如果這個使用者不能更新貼文就...
    }

<a name="creating-policies"></a>
## 建立原則

<a name="generating-policies"></a>
### 產生原則

「原則」是組織一個採用程式邏輯去區分它們特定的模型或資源之授權類別。舉例來說，如果你的應用程式是部落格，你可能會有一個 `Post` 模型和一個相對應的 `PostPolicy` 來授權使用者行為，像是建立新文章或是更新文章。

你可以使用 `make:policy` [artisan command](/docs/{{version}}/artisan)來產生一個原則。這個產生的原則將會在 `app/Policies` 這個目錄。如果這個目錄不存在， Laravel 會幫你建立唷：

    php artisan make:policy PostPolicy

`make:policy` 這個指令將會產生一個空的原則類別。如果你需要產生一個俱有基本「 CRUD 」的原則方法在類別裡面，你可以在執行指令時，後面加入 `--model` 這個選項以及參數。

    php artisan make:policy PostPolicy --model=Post

> {溫馨提醒} 全部的原則都會透過 Laravel [服務容器](/docs/{{version}}/container)來解析，可以讓你在建構子對任何需要的依賴使用型別提示，他們會被自動注入。

<a name="registering-policies"></a>
### 註冊原則

一旦原則存在，它就需要被註冊。新的 Laravel 應用程式已內建一個 `AuthServiceProvider`提供者，且包含一個 `policies` 屬性，會測繪（ maps ）你的 Eloquent 模型去比對它們的原則。註冊原則將會告訴 Laravel 在授權行為存取特定模型的過程中應該使用哪一個原則：

    <?php

    namespace App\Providers;

    use App\Post;
    use App\Policies\PostPolicy;
    use Illuminate\Support\Facades\Gate;
    use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

    class AuthServiceProvider extends ServiceProvider
    {
        /**
         * The policy mappings for the application.
         *
         * @var array
         */
        protected $policies = [
            Post::class => PostPolicy::class,
        ];

        /**
         * 註冊任何認證或授權的服務
         *
         * @return void
         */
        public function boot()
        {
            $this->registerPolicies();

            //
        }
    }

<a name="writing-policies"></a>
## 撰寫原則

<a name="policy-methods"></a>
### 原則方法

一旦原則被註冊，你就可以為授權的每個動作添加方法。舉例來說，讓我們在 `PostPolicy` 定義一個 `update` 方法，它會判斷給定的 `User` 使否可以更新一筆 `Post`。

`update` 方法會接收 `User` 和 `Post` 的實例作為參數，還會根據使否被授權更新給定的 `Post` 來回傳 `true` 或 `false`。 所以，底下的例子，讓我們驗證使用者的 `id` 是否與 Post 的 `user_id` 匹配：

    <?php

    namespace App\Policies;

    use App\User;
    use App\Post;

    class PostPolicy
    {
        /**
         * 確認使用者是否有被授權可以更新這篇貼文
         *
         * @param  \App\User  $user
         * @param  \App\Post  $post
         * @return bool
         */
        public function update(User $user, Post $post)
        {
            return $user->id === $post->user_id;
        }
    }

你可以接著此原則定義額外的方法，作為各種行為需要的授權。舉例來說，你可以定義 `view` 或 `delete` 方法來授權 `Post` 的其他行為。但請記得，你可以任意地給原則方法取自己想要的名字。

> {溫馨提醒} 當你使用 Artisan 終端指令產生你的原則時，如果使用 `--model` 選項，它會包含 `view` 、 `create` 、 `update` 和 `delete` 這些行為。

<a name="methods-without-models"></a>
### 無法核對授權資料的情況

一些原則方法只有接收當前驗證的使用者，而不再接收他們授權的模型實例。這個情況很常發生在授權 `create` 行為的時候。舉例來說，如果你正在建立一個部落格，你可能希望檢查是否有授權可以建立任何文章的權限給使用者。

當定義一個不需要接收模型實例的原則方法時，比如 `create` 方法，它不用接收模型實例。相反的，你應該將該方法定義為只要認證的使用者就好：
    /**
     * 確認使用者是否有被授權可以更新這篇貼文
     *
     * @param  \App\User  $user
     * @return bool
     */
    public function create(User $user)
    {
        //
    }

<a name="policy-filters"></a>
### 原則篩選器

對於特定使用者，你可能希望授權原則上的全部行為。要達成此目的，請在原則上定義 `before` 方法。 `before` 方法會優先執行於原則上的其他方法，在原則上的其他方法執行之前，讓你有辦法授權一些特定的權限。這個功能最常用在授權給後台管理員執行任何操作：

    public function before($user, $ability)
    {
        if ($user->isSuperAdmin()) {
            return true;
        }
    }

如果你想拒絕使用者的所有授權，你應當再 `before` 方法回傳 `false`。如果回傳 `null` ，則由原則的其他方法判斷使用者授權情況。

<a name="authorizing-actions-using-policies"></a>
## 使用原則的授權行為

<a name="via-the-user-model"></a>
### 透過使用者模型

Laravel 內建的 `User` 模型有包含兩個輔助方法來授權行為： `can` 和 `cant`。 `can` 方法專門接收你想要授權的行為和模型關聯。舉例來說：讓我們確定使用者使否有被授權更新給定的 `Post` 模型：

    if ($user->can('update', $post)) {
        //
    }

如果給定的模型的[原則已被註冊](#registering-policies)， `can` 方法將會自動呼叫適合的原則並回傳布林值結果。如果沒有為模型註冊任何原則， `can` 方法將會嘗試呼叫基於閉包的 Gate 來核對給定行為的名稱。

#### 不需指定模型的操作

請記得，有些像是 `create` 的行為不需要指定模型實例。在這些情況下，你可以傳遞一個類別名稱給 `can` 方法。這個類別名稱將會用來確定授權時該用哪個原則：

    use App\Post;

    if ($user->can('create', Post::class)) {
        // 在相關原則上執行 `create` 方法...
    }

<a name="via-middleware"></a>
### 透過中介層

Laravel 有一個可以再傳入請求到你的路由或控制器前就授權行為的中介層。預設的情況下， `Illuminate\Auth\Middleware\Authorize` 這個中介層被指定到 `App\Http\Kernel` 類別的 `can` 鍵上。讓我們來看一下使用 `can` 中介層來授權使用者更新部落格文章的例子：

    use App\Post;

    Route::put('/post/{post}', function (Post $post) {
        // 當前使用者可以更新貼文...
    })->middleware('can:update,post');

在這個例子中，我們傳遞兩個參數給 `can` 中介層。第一個是我們希望授權的行為名稱，第二個是我們希望傳遞這個原則方法的路由參數。在這個情況下，因為我們使用了[隱式模型綁定](/docs/{{version}}/routing#implicit-binding)，一個 `Post` 模型將會被傳遞給原則方法。如果使用者沒有被授權給定的行為，這個中介層將回應 HTTP `403` 的狀態碼。

#### 不需指定模型的操作

再次提醒，有些像是 `create` 的行為不需要指定模型實例。在這些情況下，你可以傳遞一個類別名稱到中介層。這個類別名稱將會用來確定授權行為時應該用哪個原則。

    Route::post('/post', function () {
        // 當前使用者可以建立新文章...
    })->middleware('can:create,App\Post');

<a name="via-controller-helpers"></a>
### 透過控制器的輔助函式

除了在 `User` 模型提供輔助方法外， Laravel 為所有繼承 `App\Http\Controllers\Controller` 的控制器提供了一個 `authorize` 輔助方法。像是 `can` 方法，這個方法接受你需要授權的行為名稱和模型關聯。如果這個行為沒被授權， `authorize` 方法會拋出 `Illuminate\Auth\Access\AuthorizationException`， Laravel 預設會使用例外處理器回應 HTTP 403 狀態碼：

    <?php

    namespace App\Http\Controllers;

    use App\Post;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class PostController extends Controller
    {
        /**
         * 更新給定的部落格貼文.
         *
         * @param  Request  $request
         * @param  Post  $post
         * @return Response
         */
        public function update(Request $request, Post $post)
        {
            $this->authorize('update', $post);

            // 當前使用者能更新部落格貼文...
        }
    }

#### 不需要指定模型的操作

如前所述，有些像是 `create` 的行為不需要指定模型實例。在這種情況下，你可以傳遞類別名稱到 `authorize` 方法。這個類別名稱將會用來確定授權行為時應該用哪個原則：

    /**
     * 建立新的貼文
     *
     * @param  Request  $request
     * @return Response
     */
    public function create(Request $request)
    {
        $this->authorize('create', Post::class);

        // 當前使用者可以建立貼文...
    }

<a name="via-blade-templates"></a>
### 透過 Blade 模板

當在撰寫 Blade 模板時，你可能希望頁面的一部分只顯示給又被授權給定行為的使用者。舉例來說，你可能希望只讓權限的使用者可以看到部落格貼文的「編輯」按鈕。在這個情況，你可以使用 `@can` 和 `@cannot` 的模板指令：

    @can('update', $post)
        <!-- 當前使用者能更新這篇貼文 -->
    @elsecan('create', $post)
        <!-- 當前使用者可以建立新貼文 -->
    @endcan

    @cannot('update', $post)
        <!-- 當前使用者不能更新這篇貼文 -->
    @elsecannot('create', $post)
        <!-- 當前使用者不能建立新貼文 -->
    @endcannot

這些指令在撰寫 `@if` 和 `@unless` 語句的縮短寫法。上面的 `@can` 和 `@cannot` 語句將分別轉換為以下語句：

    @if (Auth::user()->can('update', $post))
        <!-- 當前使用者能更新這篇貼文 -->
    @endif

    @unless (Auth::user()->can('update', $post))
        <!-- 當前使用者不能更新這篇貼文 -->
    @endunless

#### 不需要指定模型的操作

像其他授權一樣，如果這個行為不能請求一個模型實例，你可以傳遞一個類別名稱到 `@can` 和 `@cannot` 指令：

    @can('create', App\Post::class)
        <!-- 當前使用者可以建立新貼文 -->
    @endcan

    @cannot('create', App\Post::class)
        <!-- 當前使用者不能建立新貼文 -->
    @endcannot
