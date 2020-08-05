# 驗證

- [介紹](#introduction)
- [驗證快速上手](#validation-quickstart)
    - [定義路由](#quick-defining-the-routes)
    - [建立控制器](#quick-creating-the-controller)
    - [撰寫驗證邏輯](#quick-writing-the-validation-logic)
    - [顯示驗證錯誤](#quick-displaying-the-validation-errors)
    - [關於可選欄位的說明](#a-note-on-optional-fields)
- [表單請求驗證](#form-request-validation)
    - [建立表單請求](#creating-form-requests)
    - [授權表單請求](#authorizing-form-requests)
    - [自訂錯誤訊息](#customizing-the-error-messages)
    - [自訂驗證屬性](#customizing-the-validation-attributes)
    - [在驗證輸入之前](#prepare-input-for-validation)
- [手動建立驗證器](#manually-creating-validators)
    - [自動重導](#automatic-redirection)
    - [命名錯誤清單](#named-error-bags)
    - [驗證後的掛勾](#after-validation-hook)
- [處理錯誤訊息](#working-with-error-messages)
    - [自訂錯誤訊息](#custom-error-messages)
- [可用的驗證規則](#available-validation-rules)
- [依條件增加規則](#conditionally-adding-rules)
- [驗證陣列](#validating-arrays)
- [自訂驗證規則](#custom-validation-rules)
    - [使用規則物件](#using-rule-objects)
    - [使用閉包](#using-closures)
    - [使用擴充功能](#using-extensions)    
    - [隱式擴充功能](#implicit-extensions)

<a name="introduction"></a>
## 介紹

Laravel 提供多種方法來驗證應用程式傳入的資料。預設情況下，Laravel 的基底控制器利用 `ValidatesRequests` trait 提供的一個便利的方法和各種強大的驗證規則來驗證傳入的 HTTP 請求。

<a name="validation-quickstart"></a>
## 驗證快速上手

要了解 Laravel 強大的驗證特性，讓我們來看一個驗證表單並顯示錯誤訊息給使用者的完整範例。

<a name="quick-defining-the-routes"></a>
### 定義路由

首先，假設我們在 `routes/web.php` 檔案中定義了下列的路由：

    Route::get('post/create', 'PostController@create');

    Route::post('post', 'PostController@store');

`GET` 路由會顯示讓使用者新增部落格文章的表單，而 `POST` 路由會儲存部落格新文章到資料庫。

<a name="quick-creating-the-controller"></a>
### 建立控制器

再來，我們來看看一個處理這些路由的簡單的控制器。暫時先把 `store` 方法的內容留白：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;

    class PostController extends Controller
    {
        /**
         * 顯示建立部落格新文章的表單。
         *
         * @return Response
         */
        public function create()
        {
            return view('post.create');
        }

        /**
         * 儲存一篇部落格新文章。
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // 驗證並儲存部落格文章...
        }
    }

<a name="quick-writing-the-validation-logic"></a>
### 撰寫驗證邏輯

現在我們可以把驗證部落格新文章的邏輯寫進 `store` 方法中了。我們會使用 `Illuminate\Http\Request` 物件提供的 `validate` 方法來實現驗證。如果通過驗證規則，程式會繼續正常執行；如果驗證失敗，會拋出一個例外並把適當的錯誤訊息回傳給使用者。在傳統的 HTTP 請求中，會產生一個重導回應，對於 AJAX 請求則發送 JSON 回應。

為了更理解 `validate` 方法，我們先回到 `store` 方法中：

    /**
     * 儲存一篇部落格新文章。
     *
     * @param  Request  $request
     * @return Response
     */
    public function store(Request $request)
    {
        $validatedData = $request->validate([
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ]);

        // 部落格文章通過驗證...
    }

如你所見，我們單純的把所需的驗證規則傳進 `validate` 方法中。再次提醒，如果驗證失敗，會自動產生適當的回應。如果驗證通過，控制器會繼續正常執行。

除此之外，驗證規則可以是規則陣列，而不是單個 `|` 分隔字串：

    $validatedData = $request->validate([
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);

你可以使用 `validateWithBag` 方法來驗證請求並記錄任何錯誤到[命名錯誤清單](#named-error-bags):

    $validatedData = $request->validateWithBag('post', [
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);

#### 在首次驗證失敗時停止驗證

有時候你會希望在一個屬性首次驗證失敗時，停止此屬性的其他驗證規則。把 `bail` 規則指派到屬性中來達成目的：

    $request->validate([
        'title' => 'bail|required|unique:posts|max:255',
        'body' => 'required',
    ]);

在這個範例中，如果 `title` 屬性的 `unique` 規則驗證失敗，就不會檢查 `max` 規則。驗證的順序會依照規則指派的順序。

#### 關於巢狀屬性的提醒

如果 HTTP 請求包含了「巢狀」參數，可以在驗證規則中使用「點」語法來指定屬性：

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'author.name' => 'required',
        'author.description' => 'required',
    ]);

<a name="quick-displaying-the-validation-errors"></a>
### 顯示驗證錯誤

那麼當傳入的要求參數沒有通過指定的驗證規則呢？如之前提到，Laravel 會自動把使用者導回先前的位置。另外，所有的驗證錯誤會自動[快閃到 session](/docs/{{version}}/session#flash-data)。

注意到我們在 `GET` 路由中不需要明確地綁定錯誤訊息到視圖。這是因為 Laravel 會自動檢查 session 內的錯誤資料，如果錯誤存在的話，會自動綁定這些錯誤訊息到視圖。`$errors` 變數會是 `Illuminate\Support\MessageBag` 的實例。有關此物件的詳細資訊，[請查閱它的文件](#working-with-error-messages)。

> {tip} `$errors` 變數透過 `web` 中介層群組提供的 `Illuminate\View\Middleware\ShareErrorsFromSession` 中介層綁定到視圖。**當應用這個中介層時，視圖中會永遠存在一個可用的 `$errors` 變數**，你可以方便的假設 `$errors` 變數總是有被定義且可以安全使用。

在我們的範例中，驗證失敗時會將使用者重導到控制器的 `create` 方法，讓我們可以在這個視圖中顯示錯誤訊息：

    <!-- /resources/views/post/create.blade.php -->

    <h1>新增文章</h1>

    @if ($errors->any())
        <div class="alert alert-danger">
            <ul>
                @foreach ($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
            </ul>
        </div>
    @endif

    <!-- 新增文章表單 -->

#### `@error` 語法糖

你可以使用 `@error` [Blade](/docs/{{version}}/blade) 語法糖來快速檢查給定的屬性是否有錯誤訊息。可以在 `@error` 語法糖內呼叫 `$message` 變數來顯示錯誤訊息：

    <!-- /resources/views/post/create.blade.php -->

    <label for="title">Post Title</label>

    <input id="title" type="text" class="@error('title') is-invalid @enderror">

    @error('title')
        <div class="alert alert-danger">{{ $message }}</div>
    @enderror

<a name="a-note-on-optional-fields"></a>
### 關於可選欄位的說明

Laravel 預設會在全域的中介層堆疊中加入 `TrimStrings` 和 `ConvertEmptyStringsToNull` 中介層。`App\Http\Kernel` 類別中列出了堆疊內的中介層。因此，如果你不想讓驗證器認為 `null` 值無效，你通常會需要把「可選」的請求欄位標註為 `nullable`。例如：

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

在這個範例中，我們指定 `publish_at` 欄位可以為 `null` 或一個有效的日期表示。如果沒有把 `nullable` 修飾字加到規則定義中，驗證器會認為 `null` 是無效的日期。

<a name="quick-ajax-requests-and-validation"></a>
#### AJAX 請求和驗證

在這個範例中，我們使用傳統的表單來發送資料給應用程式。然而，許多應用程式是利用 AJAX 請求。在 AJAX 請求時使用 `validate` 方法，Laravel 不會產生重導回應。而是會產生一個包含所有驗證錯誤的 JSON 回應。此 JSON 回應會包含 422 HTTP 狀態碼。

<a name="form-request-validation"></a>
## 表單請求驗證

<a name="creating-form-requests"></a>
### 建立表單請求

在更複雜的驗證情境中，你可能會想建立一個「表單請求」。表單請求是一種自訂的請求類別，其中包含驗證邏輯。使用 Artisan 命令列指令 `make:request` 來建立一個表單請求類別：

    php artisan make:request StoreBlogPost

新產生的類別檔會放在 `app/Http/Requests` 目錄下。如果目錄不存在，會在執行 `make:request` 時建立。讓我們加入一些驗證規則到 rules 方法中：

    /**
     * 取得適用於請求的驗證規則。
     *
     * @return array
     */
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ];
    }

> {tip} 你可以在 `rules` 方法參數中加入任何你需要的依賴項目。他們會自動透過 Laravel [服務容器]](/docs/{{version}}/container)來解析。

那驗證的規則會如何執行呢？只需要在控制器方法中，為請求加上型別提示。傳入的表單請求會在控制器方法被呼叫前進行驗證，也就是說不會因為驗證邏輯而把控制器弄得一團亂：

    /**
     * 儲存傳入的部落格文章。
     *
     * @param  StoreBlogPost  $request
     * @return Response
     */
    public function store(StoreBlogPost $request)
    {
        // 有效的傳入請求...

        // 取得已驗證過的資料...
        $validated = $request->validated();
    }

如果驗證失敗，會產生一個重導回應把使用者導回先前的位置。這些錯誤會被快閃至 session，讓錯誤都可以被顯示。如果是 AJAX 請求，則會傳回包含 422 狀態碼及驗證錯誤的 JSON 資料的 HTTP 回應。

#### 為表單請求加上驗證後的掛勾

如果想要為表單請求加上「驗證後」的掛勾，可以利用 `withValidator` 方法。這個方法接收完整建構的驗證器，讓你可以在驗證規則實際被執行前呼叫驗證器的任何方法：

    /**
     * 設定驗證器實例。
     *
     * @param  \Illuminate\Validation\Validator  $validator
     * @return void
     */
    public function withValidator($validator)
    {
        $validator->after(function ($validator) {
            if ($this->somethingElseIsInvalid()) {
                $validator->errors()->add('field', 'Something is wrong with this field!');
            }
        });
    }

<a name="authorizing-form-requests"></a>
### 授權表單請求

表單請求類別也包含了 `authorize` 方法。在這個方法中，你可以確認使用者是否真的有權限可以更新特定資料。舉個例子，當一個使用者嘗試更新部落格文章的評論時，可以先判斷這是否是他的評論？例如：

    /**
     * 判斷使用者是否有權限做出此請求。
     *
     * @return bool
     */
    public function authorize()
    {
        $comment = Comment::find($this->route('comment'));

        return $comment && $this->user()->can('update', $comment);
    }

因為所有的表單請求都是繼承 Laravel 基底的請求類別，我們可以利用 `user` 方法來存取當下認證的使用者。同時注意到上面範例中有呼叫 `route` 方法。這個方法讓你可以取得呼叫路由時的 URI 參數，像是如下範例的 `{comment}` 參數：

    Route::post('comment/{comment}');

如果 `authorize` 方法回傳 `false`，會自動回傳一個包含 403 狀態碼的 HTTP 回應，且不會執行控制器方法。

如果你打算在應用程式的其他部分處理授權邏輯，只需讓 `authorize` 方法回傳 `true`：

    /**
     * 判斷使用者是否有權限做出此請求。
     *
     * @return bool
     */
    public function authorize()
    {
        return true;
    }

> {tip} 你可以在 `authorize` 方法參數中加入任何你需要的依賴項目。他們會自動透過 Laravel [服務容器]](/docs/{{version}}/container)來解析。

<a name="customizing-the-error-messages"></a>
### 自訂錯誤訊息

你可以透過覆寫表單請求的 `messages` 方法來自定錯誤訊息。此方法必須回傳一個包含成對的屬性與規則及對應錯誤訊息的陣列：

    /**
     * 取得已定義驗證規則的錯誤訊息。
     *
     * @return array
     */
    public function messages()
    {
        return [
            'title.required' => 'A title is required',
            'body.required' => 'A message is required',
        ];
    }

<a name="customizing-the-validation-attributes"></a>
### 自訂驗證屬性

如果想要自訂屬性名稱來替換驗證訊息的 `:attribute` 部分，你可以透過覆寫 `attributes` 方法來指定自訂的名稱。這個方法會回傳一組屬性與名稱的陣列：

    /**
     * 取得自訂的驗證器錯誤屬性。
     *
     * @return array
     */
    public function attributes()
    {
        return [
            'email' => 'email address',
        ];
    }

<a name="prepare-input-for-validation"></a>
### 在驗證輸入之前

如果你需要在驗證規則之前先修飾請求中的任何資料。你能用 `prepareForValidation` 方法：

    use Illuminate\Support\Str;

    /**
     * 在驗證輸入之前。
     *
     * @return void
     */
    protected function prepareForValidation()
    {
        $this->merge([
            'slug' => Str::slug($this->slug),
        ]);
    }

<a name="manually-creating-validators"></a>
## 手動建立驗證器

如果你不想要用請求的 `validate` 方法，你可以透過 `Validator` [facade](/docs/{{version}}/facades) 來建立驗證器實例。Facade 中的 `make` 方法會產生一個新的驗證器實例。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Validator;

    class PostController extends Controller
    {
        /**
         * 儲存部落格新文章。
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            $validator = Validator::make($request->all(), [
                'title' => 'required|unique:posts|max:255',
                'body' => 'required',
            ]);

            if ($validator->fails()) {
                return redirect('post/create')
                            ->withErrors($validator)
                            ->withInput();
            }

            // 儲存部落格文章...
        }
    }

傳給 `make` 方法的第一個參數是需要被驗證的資料。第二個參數是資料使用的驗證規則。

檢查請求是否有驗證通過後，你可以使用 `withErrors` 方法把錯誤訊息快閃到 session。使用這個方法，在重導之後 `$errors` 變數可以自動的在視圖中共用，讓你輕鬆地顯示這些訊息給使用者。`withErrors` 方法接受驗證器、`MessageBag`，或是 PHP `array`。

<a name="automatic-redirection"></a>
### 自動重導

如果在手動建立驗證器的同時，仍然想利用請求的 `validate` 方法所提供的自動重導，你可以呼叫現有的驗證器實體的 `validate` 方法。如果驗證失敗，使用者會被自動重導，或在 AJAX 請求時回傳 JSON 回應：

    Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validate();

你可以使用 `validateWithBag` 方法在驗證失敗的時候將錯誤訊息記錄到[命名錯誤清單](#named-error-bags)：

    Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validateWithBag('post');

<a name="named-error-bags"></a>
### 命名錯誤清單

假如在一個頁面中有多個表單，你可能希望為每個錯誤的 `MessageBag` 命名，這樣就可以取得特定表單的錯誤訊息。只要在 `withErrors` 的第二個參數傳入名稱即可：

    return redirect('register')
                ->withErrors($validator, 'login');

再來就可以從 `$errors` 變數中取得已命名的 `MessageBag` 實例：

    {{ $errors->login->first('email') }}

<a name="after-validation-hook"></a>
### 驗證後的掛勾

驗證器可以附加回呼以在驗證完成後執行。可以讓你更簡單的做進一步驗證甚至在錯誤訊息集合中增加更多錯誤訊息。在驗證器實例使用 `after` 方法即可：

    $validator = Validator::make(...);

    $validator->after(function ($validator) {
        if ($this->somethingElseIsInvalid()) {
            $validator->errors()->add('field', 'Something is wrong with this field!');
        }
    });

    if ($validator->fails()) {
        //
    }

<a name="working-with-error-messages"></a>
## 處理錯誤訊息

呼叫 `Validator` 實例的 `errors` 方法，會得到一個 `Illuminate\Support\MessageBag` 的實例，其中有許多方便的方法讓你處理錯誤訊息。在所有視圖中共用的 `$errors` 變數就是 `MessageBag` 類別的實例。

#### 取出特定欄位的第一個錯誤訊息

使用 `first` 方法來取出特定欄位的第一個錯誤訊息

    $errors = $validator->errors();

    echo $errors->first('email');

#### 取出特定欄位的所有錯誤訊息

使用 `get` 方法來取出特定欄位所有訊息的陣列：

    foreach ($errors->get('email') as $message) {
        //
    }

如果你驗證的表單欄位是個陣列，可以用 `*` 字元來取出每個陣列元素的訊息：

    foreach ($errors->get('attachments.*') as $message) {
        //
    }

#### 取出所有欄位的所有錯誤訊息

使用 `all` 方法來取出所有欄位的所有錯誤訊息：

    foreach ($errors->all() as $message) {
        //
    }

#### 判斷特定欄位是否有錯誤訊息

使用 `has` 方法來判斷特定欄位是否有錯誤訊息：

    if ($errors->has('email')) {
        //
    }

<a name="custom-error-messages"></a>
### 自訂錯誤訊息

如果有需要，你可以自訂驗證的錯誤訊息來取代預設的錯誤訊息。有數種指定自訂訊息的方法。第一種，把自訂的訊息傳到 `Validator::make` 方法的第三個參數：

    $messages = [
        'required' => 'The :attribute field is required.',
    ];

    $validator = Validator::make($input, $rules, $messages);

在這個範例中，`:attribute` 佔位符會被驗證時實際的欄位名稱給取代。也有其他的佔位符可以在驗證訊息中使用。例如：

    $messages = [
        'same' => ':attribute 和 :other 必須相同。',
        'size' => ':attribute 的大小必須是 :size。',
        'between' => ':attribute 必須介於 :min - :max 之間。',
        'in' => ':attribute 必須是以下的類型之一：:values',
    ];

#### 指定自訂訊息給特定的屬性

有時候你可能想要對特定的欄位自訂錯誤訊息。在你的屬性名稱後加上「.」符號，並加上指定的驗證規則：

    $messages = [
        'email.required' => '我們需要知道你的 e-mail 位址！',
    ];

<a name="localization"></a>
#### 在語系檔中指定自訂訊息

在大部分情況下，你可能會希望在語系檔中指定自訂的訊息，而不是被直接傳進 `Validator`。把訊息加到 `resources/lang/xx/validation.php` 語言檔中的 `custom` 陣列來達成目的。

    'custom' => [
        'email' => [
            'required' => '我們需要知道你的 e-mail 位址！',
        ],
    ],

#### 指定自訂的屬性值

若想要自訂屬性名稱來替換驗證訊息的 `:attribute` 部分，你可以在 `resources/lang/xx/validation.php` 語系檔的 `attributes` 陣列中指定自訂名稱：

    'attributes' => [
        'email' => 'email 位置',
    ],

你也可以傳入自訂屬性到 `Validator::make` 方法的第四個參數：

    $customAttributes = [
        'email' => 'email 位置',
    ];

    $validator = Validator::make($input, $rules, $messages, $customAttributes);

#### 在語系檔中指定自訂值

有時你需要將驗證訊息的 `:value` 部分換成自訂的值。例如，下列示範為如果 `payment_type` 的值為 `cc` 則需要信用卡號：

    $request->validate([
        'credit_card_number' => 'required_if:payment_type,cc'
    ]);

如果驗證該規則失敗，就會產生以下錯誤訊息：

    The credit card number field is required when payment type is cc.

`cc` 除了作為付款類型的值來顯示外，你可以在 `validation` 語系檔中的 `values` 陣列來定義特定的值來表示：

    'values' => [
        'payment_type' => [
            'cc' => '信用卡'
        ],
    ],

如果現在驗證失敗時，則會產生以下訊息：

    The credit card number field is required when payment type is 信用卡.

<a name="available-validation-rules"></a>
## 可用的驗證規則

以下是所有可用的驗證規則與功能：

<style>
    .collection-method-list > p {
        column-count: 3; -moz-column-count: 3; -webkit-column-count: 3;
        column-gap: 2em; -moz-column-gap: 2em; -webkit-column-gap: 2em;
    }

    .collection-method-list a {
        display: block;
    }
</style>

<div class="collection-method-list" markdown="1">

[Accepted](#rule-accepted)
[Active URL](#rule-active-url)
[After (Date)](#rule-after)
[After Or Equal (Date)](#rule-after-or-equal)
[Alpha](#rule-alpha)
[Alpha Dash](#rule-alpha-dash)
[Alpha Numeric](#rule-alpha-num)
[Array](#rule-array)
[Bail](#rule-bail)
[Before (Date)](#rule-before)
[Before Or Equal (Date)](#rule-before-or-equal)
[Between](#rule-between)
[Boolean](#rule-boolean)
[Confirmed](#rule-confirmed)
[Date](#rule-date)
[Date Equals](#rule-date-equals)
[Date Format](#rule-date-format)
[Different](#rule-different)
[Digits](#rule-digits)
[Digits Between](#rule-digits-between)
[Dimensions (Image Files)](#rule-dimensions)
[Distinct](#rule-distinct)
[Email](#rule-email)
[Ends With](#rule-ends-with)
[Exclude If](#rule-exclude-if)
[Exclude Unless](#rule-exclude-unless)
[Exists (Database)](#rule-exists)
[File](#rule-file)
[Filled](#rule-filled)
[Greater Than](#rule-gt)
[Greater Than Or Equal](#rule-gte)
[Image (File)](#rule-image)
[In](#rule-in)
[In Array](#rule-in-array)
[Integer](#rule-integer)
[IP Address](#rule-ip)
[JSON](#rule-json)
[Less Than](#rule-lt)
[Less Than Or Equal](#rule-lte)
[Max](#rule-max)
[MIME Types](#rule-mimetypes)
[MIME Type By File Extension](#rule-mimes)
[Min](#rule-min)
[Not In](#rule-not-in)
[Not Regex](#rule-not-regex)
[Nullable](#rule-nullable)
[Numeric](#rule-numeric)
[Password](#rule-password)
[Present](#rule-present)
[Regular Expression](#rule-regex)
[Required](#rule-required)
[Required If](#rule-required-if)
[Required Unless](#rule-required-unless)
[Required With](#rule-required-with)
[Required With All](#rule-required-with-all)
[Required Without](#rule-required-without)
[Required Without All](#rule-required-without-all)
[Same](#rule-same)
[Size](#rule-size)
[Sometimes](#conditionally-adding-rules)
[Starts With](#rule-starts-with)
[String](#rule-string)
[Timezone](#rule-timezone)
[Unique (Database)](#rule-unique)
[URL](#rule-url)
[UUID](#rule-uuid)

</div>

<a name="rule-accepted"></a>
#### accepted

驗證欄位值是否為 _yes_、_on_、_1_、或 _true_。這在確認是否同意「服務條款」時很有用。

<a name="rule-active-url"></a>
#### active_url

透過 PHP 的 `dns_get_record` 函式來驗證欄位是否為有效的 A 或 AAAA 紀錄。會在開始傳入 `dns_get_record` 之前使用 PHP 的 `parse_url` 函數來取得提供 URL 的主機名稱。

<a name="rule-after"></a>
#### after:_date_

驗證欄位是否是在指定日期之後的值。這個日期會透過 PHP `strtotime` 函式驗證。

    'start_date' => 'required|date|after:tomorrow'

或者指定另一個用來比較的日期欄位，而不是透過傳給 `strtotime` 的字串：

    'finish_date' => 'required|date|after:start_date'

<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_

驗證欄位是否在指定日期之後或同一天。更多資訊請參考 [after](#rule-after) 規則。

<a name="rule-alpha"></a>
#### alpha

驗證欄位值是否僅包含字母及數字。

<a name="rule-alpha-dash"></a>
#### alpha_dash

驗證欄位值是否僅包含字母、數字、破折號（ - ）以及底線（ _ ）。

<a name="rule-alpha-num"></a>
#### alpha_num

驗證欄位值是否僅包含字母及數字。

<a name="rule-array"></a>
#### array

驗證欄位必須是一個 PHP `array`。

<a name="rule-bail"></a>
#### bail

在第一次驗證失敗後，停止繼續驗證規則。

<a name="rule-before"></a>
#### before:_date_

驗證欄位是否是在指定日期之前。這個日期會透過 PHP `strtotime` 函式驗證。此外，可以像 [`after`](#rule-after) 規則一樣，將另一個正在驗證的欄位名稱作為 `date` 的值來驗證。

<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

驗證欄位是否在指定日期之前或同一天。這個日期會透過 PHP `strtotime` 函式驗證。可以像 [`after`](#rule-after) 規則一樣，將另一個正在驗證的欄位名稱作為 `date` 的值來驗證。

<a name="rule-between"></a>
#### between:_min_,_max_

驗證欄位值的大小是否介於指定的 _min_ 和 _max_ 之間。字串、數值、陣列和檔案大小的計算方式和 [`size`](#rule-size) 規則相同。

<a name="rule-boolean"></a>
#### boolean

驗證欄位值必須能夠轉型為布林值。可接受的輸入為 `true`、`false`、`1`、`0`、`"1"` 以及 `"0"`。

<a name="rule-confirmed"></a>
#### confirmed

驗證欄位值必須和對應的 `foo_confirmation` 欄位值相同。例如，如果要驗證 `password` 欄位，必須和輸入資料裡的 `password_confirmation` 欄位值相同。

<a name="rule-date"></a>
#### date

驗證欄位值是有效的日期，會透過 PHP 的 `strtotime` 函式驗證。

<a name="rule-date-equals"></a>
#### date_equals:_date_

驗證欄位是否與指定日期同一天。這個日期會透過 PHP `strtotime` 函式驗證。

<a name="rule-date-format"></a>
#### date_format:_format_

驗證欄位值符合定義的日期格式（ _format_ ）。你應該使用 `date` 或 `date_format` **其一**來驗證欄位，而不是兩者。這個驗證規則支援 PHP 的 [DateTime](https://www.php.net/manual/en/class.datetime.php) 類別的所有格式。

<a name="rule-different"></a>
#### different:_field_

驗證欄位值是否和指定的欄位（ _field_ ）不同。

<a name="rule-digits"></a>
#### digits:_value_

驗證欄位值為長度為 _value_ 的_數字_。

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

驗證的欄位必須是 _numeric_，且長度必須介於給定的 _min_ 和 _max_ 之間。

<a name="rule-dimensions"></a>
#### dimensions

驗證欄位值必須是一張圖片且符合規則參數限制的尺寸：

    'avatar' => 'dimensions:min_width=100,min_height=200'

可用的限制為：_min\_width_、_max\_width_、_min\_height_、_max\_height_、_width_、_height_、_ratio_。

_ratio_ 為寬除以高的比例。可以運算式 `3/2` 或浮點數 `1.5` 來表示：

    'avatar' => 'dimensions:ratio=3/2'

由於此規則需要許多參數，你可以用 `Rule::dimensions` 方法來流暢地建構這個規則：

    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'avatar' => [
            'required',
            Rule::dimensions()->maxWidth(1000)->maxHeight(500)->ratio(3 / 2),
        ],
    ]);

<a name="rule-distinct"></a>
#### distinct

當處理陣列時，驗證欄位中不能有任何重複的值。

    'foo.*.id' => 'distinct'

<a name="rule-email"></a>
#### email

驗證欄位必須為一個 e-mail 地址。這個驗證規則會使用 [`egulias/email-validator`](https://github.com/egulias/EmailValidator) 套件來驗證 e-mail 位置。預設會使用 `RFCValidation` 驗證器，但你可以採用其他驗證器格式：

    'email' => 'email:rfc,dns'

下列範例會採用 `RFCValidation` 和 `DNSCheckValidation` 驗證器。這裡列出所有可用的驗證器格式：

<div class="content-list" markdown="1">
- `rfc`: `RFCValidation`
- `strict`: `NoRFCWarningsValidation`
- `dns`: `DNSCheckValidation`
- `spoof`: `SpoofCheckValidation`
- `filter`: `FilterEmailValidation`
</div>

`filter` 驗證器使用 PHP 的 `filter_var` 函式，與 Laravel 5.8 用法相同。`dns`和`spoof`驗證器需要 PHP `intl` 擴充功能。

<a name="rule-ends-with"></a>
#### ends_with:_foo_,_bar_,...

驗證欄位的結尾是否有給定的值。

<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

當 _anotherfield_ 欄位等於 _value_ 的時候，該驗證的欄位會從 `validate` 和 `validated` 方法回傳的請求資料中排除。

<a name="rule-exclude-unless"></a>
#### exclude_unless:_anotherfield_,_value_

當 _anotherfield_ 欄位不等於 _value_ 的時候，該驗證的欄位會從 `validate` 和 `validated` 方法回傳的請求資料中排除。

<a name="rule-exists"></a>
#### exists:_table_,_column_

驗證欄位值必須存在一個指定的資料表中。

#### Exists 規則的基本用法

    'state' => 'exists:states'

如果沒有指定 `column` 選項，會採用欄位名稱。

#### 指定欄位名稱

    'state' => 'exists:states,abbreviation'

有時候，你可能會指定 `exists` 查詢使用特定的資料庫連線。可以用「點」語法在資料表名稱前加上連線名稱來達成：

    'email' => 'exists:connection.staff,email'

可以指定 Eloquent 模型來確認資料表名稱，而不是直接指定資料表名稱：

    'user_id' => 'exists:App\User,id'

如果想要客制驗證規則執行的查詢，可以用 `Rule` 類別來流暢地定義規則。在這個範例中，我們也會以陣列來表示驗證規則，而不是用 `|` 字元來分割：

    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'email' => [
            'required',
            Rule::exists('staff')->where(function ($query) {
                $query->where('account_id', 1);
            }),
        ],
    ]);

<a name="rule-file"></a>
#### file

驗證欄位必須是一個成功上傳的檔案。

<a name="rule-filled"></a>
#### filled

當驗證欄位存在時，不可以為空值。

<a name="rule-gt"></a>
#### gt:_field_

驗證欄位必須大於給定的 _field_。驗證與被驗證的值必須是相同的型別。使用與 [`size`](#rule-size) 相同方式來驗證字串、數值、陣列和檔案。

<a name="rule-gte"></a>
#### gte:_field_

驗證欄位必須大於等於給定的 _field_。驗證與被驗證的值必須是相同的型別。使用與 [`size`](#rule-size) 相同方式來驗證字串、數值、陣列和檔案。

<a name="rule-image"></a>
#### image

驗證欄位檔案必須為圖片格式（ jpeg、png、bmp、gif、或 svg ）。

<a name="rule-in"></a>
#### in:_foo_,_bar_,...

驗證欄位必須包含在給定的列表中。由於這個規則經常需要你 `implode` 一個陣列，`Rule::in` 方法可以用來流暢地建構規則：

    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'zones' => [
            'required',
            Rule::in(['first-zone', 'second-zone']),
        ],
    ]);

<a name="rule-in-array"></a>
#### in_array:_anotherfield_.*

驗證欄位的值必須在 _anotherfield_ 陣列中存在。

<a name="rule-integer"></a>
#### integer

驗證欄位必須是一個整數。

> {note} 這個驗證規則不會驗證是否是「整數」型別，只驗證輸入的整數或字串資料是否包含數值。

<a name="rule-ip"></a>
#### ip

驗證欄位必須符合一個 IP 位址的格式。

#### ipv4

驗證欄位必須符合一個 IPv4 位址的格式。

#### ipv6

驗證欄位必須符合一個 IPv6 位址的格式。

<a name="rule-json"></a>
#### json

驗證欄位必須是一個有效的 JSON 字串。

<a name="rule-lt"></a>
#### lt:_field_

驗證欄位必須小於給定的 _field_。驗證與被驗證的值必須是相同的型別。使用與 [`size`](#rule-size) 相同方式來驗證字串、數值、陣列和檔案。

<a name="rule-lte"></a>
#### lte:_field_

驗證欄位必須小於等於給定的 _field_。驗證與被驗證的值必須是相同的型別。字串、數值和檔案大小的計算方式和 [`size`](#rule-size) 規則相同。

<a name="rule-max"></a>
#### max:_value_

驗證欄位值的大小是否小於或等於 _value_。字串、數值和檔案大小的計算方式和 [`size`](#rule-size) 規則相同。

<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

驗證檔案必須符合給定的 MIME 類型之一。

    'video' => 'mimetypes:video/avi,video/mpeg,video/quicktime'

為了判斷上傳檔案的 MIME 類型，檔案的內容會被讀取，框架會嘗試猜測 MIME 類型，這可能與客戶端提供的 MIME 類型不同。

<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

驗證檔案的 MIME 類型必須符合清單裡的副檔名之一。

#### MIME 規則基本用法

    'photo' => 'mimes:jpeg,bmp,png'

即便只需要指定副檔名，但此規則實際上透過讀取檔案內容，並猜測 MIME 類型來驗證。

完整的 MIME 類型及對應的副檔名清單可以在下方連結找到：[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="rule-min"></a>
#### min:_value_

驗證欄位值的大小是否小於或等於 _value_。字串、數值或是檔案大小的計算方式和 [`size`](#rule-size) 規則相同。

<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

驗證欄位值不在給定的清單裡。`Rule::notIn` 方法可以用來流暢地建構規則：

    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'toppings' => [
            'required',
            Rule::notIn(['sprinkles', 'cherries']),
        ],
    ]);

<a name="rule-not-regex"></a>
#### not_regex:_pattern_

驗證欄位值不能符合給定的正規表示式。

其中，這個規則有用到 PHP 的 `preg_match` 函式。選用的模式需要與 `preg_match` 使用相同的格式，因此還要有分隔符號。例如：`'email' => 'not_regex:/^.+$/i'`。

**注意:**在使用 `regex` 規則時，你可能得使用陣列而不是 `|` 分隔符號來指定規則，特別是當正規表示式含有 `|` 字元時。

<a name="rule-nullable"></a>
#### nullable

驗證欄位值可以為 `null`。在驗證可能含有 `null` 值的基本型別（如字串或整數）時特別有用。

<a name="rule-numeric"></a>
#### numeric

驗證欄位值是否為數值。

<a name="rule-password"></a>
#### password

驗證欄位必須與認證過的使用者密碼相符合，你可以使用該規則的第一個參數來指定身份認證的守衛：

    'password' => 'password:api'

<a name="rule-present"></a>
#### present

驗證欄位一定要有值，但可為空值。

<a name="rule-regex"></a>
#### regex:_pattern_

驗證欄位值符合給定的正規表示式。

其中，這個規則有用到 PHP 的 `preg_match` 函式。選用的模式需要與 `preg_match` 使用相同的格式，因此還要有分隔符號。例如：`'email' => 'not_regex:/^.+$/i'`。

**注意:**在使用 `regex` 規則時，你可能得使用陣列而不是 `|` 分隔符號來指定規則，特別是當正規表示式含有 `|` 字元時。

<a name="rule-required"></a>
#### required

驗證欄位必須有值且不為空值。欄位符合下方任一條件時為「空值」：

<div class="content-list" markdown="1">

- 該值為 `null`.
- 該值為空字串。
- 該值為空陣列或空的 `Countable` 物件。
- 該值為沒有路徑的上傳檔案。

</div>

<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

如果指定的_其它欄位（ anotherfield ）_等於任何一個 _value_ 時，此欄位必須有值且不為空值。

如果你想要為 `required_if` 規則建構更複雜的條件，你可以使用 `Rule::requiredIf`。這個方法接受布林值或閉包。若傳入的是閉包，閉包會依據回傳 的是 `true` 或 `false` 來判斷是否需要驗證欄位：

    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::requiredIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::requiredIf(function () use ($request) {
            return $request->user()->is_admin;
        }),
    ]);

<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

此欄位必須有值且不為空值，除非指定的_其它欄位（ anotherfield ）_等於任何一個 _value_。

<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

如果指定的_任一_欄位有值，則此欄位必須有值且不為空值。

<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

如果指定的_所有_欄位都有值，則此欄位必須有值且不為空值。

<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

如果_任一_指定的欄位不存在時，則此欄位必須有值且不為空值。

<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

如果_所有_指定的欄位不存在時，則此欄位必須有值且不為空值。

<a name="rule-same"></a>
#### same:_field_

驗證欄位值和指定的_欄位（ field ）_值相同。

<a name="rule-size"></a>
#### size:_value_

驗證欄位值的大小需符合給定 _value_ 值。對於字串來說，_value_ 為字元數。對於數字來說，_value_ 為指定的整數值（該屬性最好有 `numeric` 或 `integer` 規則）。對陣列來說，_size_ 對應陣列的 `count`。對檔案來說，_size_ 對應到的是檔案大小（單位 kb ）。讓我們看一些範例：

    // 驗證字串的長度為 12 字元...
    'title' => 'size:12';

    // 驗證數值等於 10...
    'seats' => 'integer|size:10';

    // 驗證陣列有五個元素...
    'tags' => 'array|size:5';

    // 驗證上傳的檔案是 512 kb...
    'image' => 'file|size:512';

<a name="rule-starts-with"></a>
#### starts_with:_foo_,_bar_,...

驗證欄位的開頭有其中一個給定的值。

<a name="rule-string"></a>
#### string

驗證欄位值的型別是否為字串。如果想允許欄位可為 `null`，可以指定 `nullable` 規則給欄位。

<a name="rule-timezone"></a>
#### timezone

驗證欄位值是否為有效的時區，會根據 PHP 的 `timezone_identifiers_list` 函式來判斷。

<a name="rule-unique"></a>
#### unique:_table_,_column_,_except_,_idColumn_

在給定的資料表，驗證欄位中必須是唯一的。如果沒有指定 `column`，將會使用欄位本身的名稱。

**指定自訂的資料表與欄位名稱：**

你可以指定 Eloquent 模型來確認資料表名稱，而不是直接指定資料表名稱：

    'email' => 'unique:App\User,email_address'

`column` 選項可被用於指定對應的資料庫欄位。如果未指定 `column` 選項，則會使用驗證的欄位名稱。

    'email' => 'unique:users,email_address'

**自訂資料庫連線**

有時候你可能需要驗證器透過自訂的連線來對資料庫進行查詢。如上面所示，設定 `unique:users` 作為驗證規則，會透過預設資料庫連線來做資料庫查詢。如果要覆寫，用「.」語法來指定連線跟資料表名稱：

    'email' => 'unique:connection.users,email_address'

**強迫 Unique 規則忽略特定 ID：**

有時候會希望在驗證欄位的時候忽略給定的 ID。例如，在「更新個人資料」時包含使用者名稱、信箱和所在區域。當然，你會想要驗證 e-mail 是否為唯一的。但如果使用者只更改名稱欄位而沒有異動 e-mail 欄位，不應該拋出驗證錯誤，因為使用者已經是這個 e-mail 的擁有者。

我們會用 `Rule` 類別來流暢地定義規則，來指示驗證器忽略使用者 ID。在這個範例中，我們同時也使用陣列而不是 `|` 字元來分隔規則：

    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'email' => [
            'required',
            Rule::unique('users')->ignore($user->id),
        ],
    ]);

> {note} 始終不該把使用者控制的請求傳入 `ignore` 方法。相對的，你應該只傳入系統產生的唯一 ID，像是從 Eloquent 模型實例自動遞增的 ID 或 UUID。否則你的應用程式將容易遭受 SQL 注入攻擊。

可以傳整個模型實例到 `ignore` 方法，而不是傳入模型的鍵值。Laravel 會自動從模型中取得這個鍵：

    Rule::unique('users')->ignore($user)

如果你的資料表的主鍵是使用 `id` 以為的欄位名稱，你可以在呼叫 `ignore` 方法時指定該欄位名稱：

    Rule::unique('users')->ignore($user->id, 'user_id')

預設的 `unique` 規則會檢查被驗證的屬性是否達到該欄位的唯一值。然而，你可以傳入不同的欄位名稱到 `unique` 方法的第二個參數：

    Rule::unique('users', 'email_address')->ignore($user->id),

**加入其他的 Where 閉包:**

你也可以使用 `where` 方法來自訂查詢指定的查詢條件。像是，加入一個驗證 `account_id` 是 `1` 的條件：

    'email' => Rule::unique('users')->where(function ($query) {
        return $query->where('account_id', 1);
    })

<a name="rule-url"></a>
#### url

驗證欄位值必須符合 URL 格式。

<a name="rule-uuid"></a>
#### uuid

驗證欄位必須是有效的 RFC 4122（版本 1、3、4 或 5）的唯一識別（UUID）。

<a name="conditionally-adding-rules"></a>
## 有條件的加入規則

#### 當欄位有特定的值時，就略過驗證

有時你會想要在另一個欄位有給定的值時不去驗證當前的欄位。可以使用 `exclude_if` 驗證規則來達到這個目的。在這個範例中，`appointment_date` 和 `doctor_name` 欄位會在 `has_appointment` 欄位的值為 `false` 時不做驗證：

    $v = Validator::make($data, [
        'has_appointment' => 'required|bool',
        'appointment_date' => 'exclude_if:has_appointment,false|required|date',
        'doctor_name' => 'exclude_if:has_appointment,false|required|string',
    ]);

相對的，你可以使用 `exclude_unless` 規則來不驗證給定的欄位，除非另一個欄位有給定的值：

    $v = Validator::make($data, [
        'has_appointment' => 'required|bool',
        'appointment_date' => 'exclude_unless:has_appointment,true|required|date',
        'doctor_name' => 'exclude_unless:has_appointment,true|required|string',
    ]);

#### 依條件增加規則

在某些情況下，你可能**只**想在輸入資料中有此欄位時才進行驗證。只要增加 `sometimes` 規則到進規則清單中，就可以快速達成：

    $v = Validator::make($data, [
        'email' => 'sometimes|required|email',
    ]);

在上面的範例中，`email` 欄位的驗證，只會在 `$data` 陣列有此欄位才會進行。

> 如果你在嘗試驗證一定會存在但可能為空值的欄位，請查看[關於可選欄位的說明](#a-note-on-optional-fields)

#### 複雜的條件驗證

有時候你可能希望以更複雜的條件邏輯來增加驗證規則。例如，你可以希望某個欄位，在另一個欄位的值超過 100 時才為必填。或者當某個指定欄位有值時，另外兩個欄位要符合特定值。增加這樣的驗證條件並不痛苦。首先，利用你熟悉的 _static rules_ 建立一個 `Validator` 實例：

    $v = Validator::make($data, [
        'email' => 'required|email',
        'games' => 'required|numeric',
    ]);

假設我們的網頁應用程式是專為遊戲收藏家所設計。如果遊戲收藏家收藏超過一百款遊戲，我們希望他們說明為什麼他們擁有這麼多遊戲。像是，可能他們經營一家二手遊戲商店，或是他們可能只是愛收集。為了在特定條件下加入此驗證需求，我們可以在 `Validator` 實例使用 `sometimes` 方法。

    $v->sometimes('reason', 'required|max:500', function ($input) {
        return $input->games >= 100;
    });

傳入 `sometimes` 方法的第一個參數，是我們要依條件驗證的欄位名稱。第二個參數是我們想加入的驗證規則。如果第三個參數傳入的`閉包`回傳 `true`，此規則就會被加入。這個方法可以輕鬆的建立複雜的條件驗證。甚至可以一次對多個欄位增加條件式驗證：

    $v->sometimes(['reason', 'cost'], 'required', function ($input) {
        return $input->games >= 100;
    });

> {tip} 傳入`閉包`的 `$input` 參數是 `Illuminate\Support\Fluent` 實例，可以用來取得輸入的資料和檔案。

<a name="validating-arrays"></a>
## 驗證陣列

驗證陣列類型的表單輸入欄位並不難。可以使用「點符號」來驗證陣列中的屬性。比如說，如果傳入的 HTTP 請求包含 `photos[profile]` 欄位，可以這樣驗證：

    $validator = Validator::make($request->all(), [
        'photos.profile' => 'required|image',
    ]);

也可以驗證陣列中的每個元素。比如這樣做可以驗證給定的陣列輸入欄位中的 e-mail 都是唯一的：

    $validator = Validator::make($request->all(), [
        'person.*.email' => 'email|unique:users',
        'person.*.first_name' => 'required_with:person.*.last_name',
    ]);

同樣也可以在語系檔中用 `*` 字元來來定義驗證訊息，輕鬆的讓陣列欄位利用單個驗證訊息：

    'custom' => [
        'person.*.email' => [
            'unique' => '每個人都要有唯一的 e-mail 位址',
        ]
    ],

<a name="custom-validation-rules"></a>
## 自訂驗證規則

<a name="using-rule-objects"></a>
### 使用規則物件

Laravel 提供多種有用的驗證規則；但你可能想要定義一些自己的規則。一種註冊自訂驗證規則的方式是使用規則物件。使用 `make:rule` Artisan 指令來產生新的規則物件。讓我們用這個指令來產生一個驗證字串是否為大寫的規則。Laravel 會把新的規則放在 `app/Rules` 目錄下：

    php artisan make:rule Uppercase

規則被建立後，我們就可以來定義它的行為。一個規則物件包含兩個方法：`passes` 和 `message`。`passes` 方法接收屬性值和名稱，並根據屬性值是否合法來回傳 `true` 或 `false`。`message` 方法應該回傳驗證失敗時使用的驗證錯誤訊息。

    <?php

    namespace App\Rules;

    use Illuminate\Contracts\Validation\Rule;

    class Uppercase implements Rule
    {
        /**
         * 判斷驗證規則是否通過。
         *
         * @param  string  $attribute
         * @param  mixed  $value
         * @return bool
         */
        public function passes($attribute, $value)
        {
            return strtoupper($value) === $value;
        }

        /**
         * 取得驗證錯誤訊息。
         *
         * @return string
         */
        public function message()
        {
            return 'The :attribute must be uppercase.';
        }
    }

如果想要從翻譯檔中取回錯誤訊息，也可以在 `message` 方法中呼叫 `trans` 輔助函式。

    /**
     * 取得驗證錯誤訊息。
     *
     * @return string
     */
    public function message()
    {
        return trans('validation.uppercase');
    }

定義好規則後，可以藉由傳遞規則物件實例來附加到其他的驗證規則中：

    use App\Rules\Uppercase;

    $request->validate([
        'name' => ['required', 'string', new Uppercase],
    ]);

<a name="using-closures"></a>
### 使用閉包

如果整個應用程式只需要使用一次的自訂規則功能，你可以使用閉包來取代規則物件。閉包會接收屬性名稱與值，還有驗證失敗後呼叫的回呼：

    $validator = Validator::make($request->all(), [
        'title' => [
            'required',
            'max:255',
            function ($attribute, $value, $fail) {
                if ($value === 'foo') {
                    $fail($attribute.' is invalid.');
                }
            },
        ],
    ]);

<a name="using-extensions"></a>
### 使用擴充功能

另一種註冊自訂驗證規則的方法是使用 `Validator` [facade](/docs/{{version}}/facades) 的 `extend` 方法。讓我們在[服務提供者](/docs/{{version}}/providers)中使用這個方法來自訂註冊的驗證規則：

    <?php

    namespace App\Providers;

    use Illuminate\Support\ServiceProvider;
    use Illuminate\Support\Facades\Validator;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 註冊服務提供者。
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * 啟動所有應用程式服務。
         *
         * @return void
         */
        public function boot()
        {
            Validator::extend('foo', function ($attribute, $value, $parameters, $validator) {
                return $value == 'foo';
            });
        }
    }

自訂的驗證閉包接收四個參數：要被驗證的屬性名稱 `$attribute`，屬性的值 `$value`，傳入驗證規則的參數陣列 `$parameters`，及 `Validator `實例。

除了使用閉包，你也可以傳入類別和方法到 `extend` 方法中：

    Validator::extend('foo', 'FooValidator@validate');

#### 定義錯誤訊息

在自訂規則中也需要定義錯誤訊息。可以使用行內的自訂訊息陣列或在驗證語系檔中加入加入一個條目。這個訊息應該被放在陣列的第一層，而不是放在對應特定屬性錯誤訊息的 `custom` 陣列：

    "foo" => "你的輸入無效！",

    "accepted" => ":attribute 必須被接受。",

    // 其餘的驗證錯誤訊息...

在建立自訂的驗證規則時，你可能需要幫錯誤訊息定義自訂的佔位符。用如上所述的方式建立自訂的驗證器後，呼叫 `Validator` facade 的 `replacer` 方法。可以在[服務提供者](/docs/{{version}}/providers)中的 `boot` 方法來做這些事：

    /**
     * 啟動所有應用程式服務。
     *
     * @return void
     */
    public function boot()
    {
        Validator::extend(...);

        Validator::replacer('foo', function ($message, $attribute, $rule, $parameters) {
            return str_replace(...);
        });
    }

<a name="implicit-extensions"></a>
### 隱式擴充功能

預設情況下，當被驗證的屬性不存在或是空字串時，則不會去執行一般和自訂擴充的驗證規則。例如，[`unique`](#rule-unique) 規則不會在空字串上執行：

    $rules = ['name' => 'unique:users,name'];

    $input = ['name' => ''];

    Validator::make($input, $rules)->passes(); // true

若要當屬性為空時依然執行該規則，那麼該規則必須暗示屬性為必填。要建立這樣的一個「隱式」擴充功能，使用 `Validator::extendImplicit()` 方法：

    Validator::extendImplicit('foo', function ($attribute, $value, $parameters, $validator) {
        return $value == 'foo';
    });

> {note} 「隱式」擴充功能只能_確保_該屬性為必填。由你決定不存在的屬性或空值是否無效。

#### 隱式規則物件

如果你想要在屬性為空值時執行規則物件，應去實作 `Illuminate\Contracts\Validation\ImplicitRule` 介面。這個介面是驗證器的「標記介面」。所以它並不會需要你實作任何方法。
