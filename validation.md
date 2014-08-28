<<<<<<< HEAD
# 驗證

- [基本用法](#basic-usage)
- [使用錯誤訊息](#working-with-error-messages)
- [錯誤訊息 & 視圖](#error-messages-and-views)
- [使用驗證規則](#available-validation-rules)
- [有條件新增規則](#conditionally-adding-rules)
- [自訂錯誤訊息](#custom-error-messages)
- [自訂驗證規則](#custom-validation-rules)

<a name="basic-usage"></a>
## 基本用法

Laravel 透過`Validation`類別讓你可以簡單，方便的驗證資料正確性及查看驗證的錯誤訊息

#### 基本驗證範例

	$validator = Validator::make(
		array('name' => 'Dayle'),
		array('name' => 'required|min:5')
	);


我們透過`make`這個方法來的第一個參數設定該被驗證資料名稱，然後第二個參數我們定義該資料可被接受的規則

#### 使用陣列來定義規則

我們可以同時定義多個規則並且使用,(逗號)分隔或是多宣告一個陣列

	$validator = Validator::make(
		array('name' => 'Dayle'),
		array('name' => array('required', 'min:5'))
	);

#### 驗證多個欄位

    $validator = Validator::make(
        array(
            'name' => 'Dayle',
            'password' => 'lamepassword',
            'email' => 'email@example.com'
        ),
        array(
            'name' => 'required',
            'password' => 'required|min:8',
            'email' => 'required|email|unique:users'
        )
    );


當一個 `Validator` 實例被建立，`fails`或`passes`這二個方法就可以在驗證時使用，如下

	if ($validator->fails())
	{
		// The given data did not pass validation
	}


假如驗證失敗，你可能需要從驗證變數$validator查看錯誤驗證可使用下列方式

	$messages = $validator->messages();


你也有可能需要透過一個陣列來了解是哪些驗證規則無法通過驗證，這個時後你可以使用`failed`這個方法：
	$failed = $validator->failed();

#### 驗證欄位

`Validator`類別提供了許多規則讓你驗證檔案，例如`size`, `mimes`,以及其他參數. 當驗證檔案時你可以簡單的透過這些參數加入你的驗證器來驗證你其他的資料.

<a name="working-with-error-messages"></a>
## 使用錯誤訊息


當你呼叫一個`Validator`實例的`messages`方法後，你會得到一個`MessageBag`的變數，裡面有許多方便的方法讓你取得錯誤訊息

#### 查看一個欄位的第一個錯誤訊息

	echo $messages->first('email');

#### 查看一個欄位的所有錯誤訊息

	foreach ($messages->get('email') as $message)
	{
		//
	}

#### 查看所有欄位的所有錯誤訊息

	foreach ($messages->all() as $message)
	{
		//
	}

#### 隔分一個欄位是否有錯誤訊息

	if ($messages->has('email'))
	{
		//
	}

#### 定義錯誤訊息格式

	echo $messages->first('email', '<p>:message</p>');

> **注意:** 預設錯誤訊息使用Bootstrap語法.

#### 查看所有錯誤訊息並使用特定格式

	foreach ($messages->all('<li>:message</li>') as $message)
	{
		//
	}

<a name="error-messages-and-views"></a>
## 錯誤訊息 & 視圖

當你開始執行驗證，你將會需要一個簡單的方法去取的錯誤訊息並回傳到你的視圖，在Laravel你可以很方便的處理這些事，你可以透過下面的路由例子來了解：


	Route::get('register', function()
	{
		return View::make('user.register');
	});

	Route::post('register', function()
	{
		$rules = array(...);

		$validator = Validator::make(Input::all(), $rules);

		if ($validator->fails())
		{
			return Redirect::to('register')->withErrors($validator);
		}
	});

記住當驗證失敗時，我們透過`Validator`的實例中`withErrors`的方法去重新導向頁面，這個方法將會存取錯誤訊息在session中，所以他們將會在下個頁面可以被使用.

僅管如此,  注意我們不需要特別去使用GET route在視圖中綁定錯誤訊息. 因為Laravel 將會一直確認seesion中的錯誤訊息並自動綁定他們在視圖當中當那些錯誤訊息是有效的時後. 

**所以這相當重要去了解，一個`$errors`變數將會永遠有效的存在你的所有的視圖中，在每個允許的頁面請求你都很直接的假設並且安全的使用`$errors`變數.這個變數將會是一個`MessageBag`的實例.

所以，當我們重新做頁面導向，你可以自然的在視圖中使用`$errors`變數

	<?php echo $errors->first('email'); ?>

### 命名錯誤清單

假如你在一個頁面中有許多的表單, 你可能希望為錯誤命名一個`MessageBag`. 這將允許你針對特定表單查看錯誤訊息, 我們只要簡單的在`withErrors`的第二個參數設定將可達到這個功能：

	return Redirect::to('register')->withErrors($validator, 'login');

你也可以接著從一個`$errors`變數中命名一個`MessageBag`實例

	<?php echo $errors->login->first('email'); ?>

<a name="available-validation-rules"></a>
## 有效的驗證規則

以下是一個有效的驗證規則清單與他們的函式

- [Accepted](#rule-accepted)
- [Active URL](#rule-active-url)
- [After (Date)](#rule-after)
- [Alpha](#rule-alpha)
- [Alpha Dash](#rule-alpha-dash)
- [Alpha Numeric](#rule-alpha-num)
- [Array](#rule-array)
- [Before (Date)](#rule-before)
- [Between](#rule-between)
- [Boolean](#rule-boolean)
- [Confirmed](#rule-confirmed)
- [Date](#rule-date)
- [Date Format](#rule-date-format)
- [Different](#rule-different)
- [Digits](#rule-digits)
- [Digits Between](#rule-digits-between)
- [E-Mail](#rule-email)
- [Exists (Database)](#rule-exists)
- [Image (File)](#rule-image)
- [In](#rule-in)
- [Integer](#rule-integer)
- [IP Address](#rule-ip)
- [Max](#rule-max)
- [MIME Types](#rule-mimes)
- [Min](#rule-min)
- [Not In](#rule-not-in)
- [Numeric](#rule-numeric)
- [Regular Expression](#rule-regex)
- [Required](#rule-required)
- [Required If](#rule-required-if)
- [Required With](#rule-required-with)
- [Required With All](#rule-required-with-all)
- [Required Without](#rule-required-without)
- [Required Without All](#rule-required-without-all)
- [Same](#rule-same)
- [Size](#rule-size)
- [Timezone](#rule-timezone)
- [Unique (Database)](#rule-unique)
- [URL](#rule-url)

<a name="rule-accepted"></a>
#### accepted

這個欄位必需是_yes_, _on_, 或 _1_時驗證才會成立. 這將會在"使用者條款"的驗證中很有用

<a name="rule-active-url"></a>
#### active_url

The field under validation must be a valid URL according to the `checkdnsrr` PHP function.
這個欄位必需是一個有效的網址並通過`checkdnsrr`這個php函式的驗證

<a name="rule-after"></a>
#### after:_date_

The field under validation must be a value after a given date. The dates will be passed into the PHP `strtotime` function.
這個欄位必需是在給允的日期後，而這個日期將會帶入`strtotime`函式進行驗證

<a name="rule-alpha"></a>
#### alpha

The field under validation must be entirely alphabetic characters.
這個欄位只能允許是字母

<a name="rule-alpha-dash"></a>
#### alpha_dash

The field under validation may have alpha-numeric characters, as well as dashes and underscores.
這個欄位只能允許字母、數字以及-及_

<a name="rule-alpha-num"></a>
#### alpha_num

The field under validation must be entirely alpha-numeric characters.
這個欄位只允許字母及數字

<a name="rule-array"></a>
#### array

The field under validation must be of type array.
這個欄位只允許陣列

<a name="rule-before"></a>
#### before:_date_

The field under validation must be a value preceding the given date. The dates will be passed into the PHP `strtotime` function.
這個欄位必需是在給允的日期早，而這個日期將會帶入`strtotime`函式進行驗證

<a name="rule-between"></a>
#### between:_min_,_max_

The field under validation must have a size between the given _min_ and _max_. Strings, numerics, and files are evaluated in the same fashion as the `size` rule.
這個欄位的大小必需介於 _min_ 及 _max_. 字串、數字、檔案的大小都是同樣依據`size`這個欄位

<a name="rule-confirmed"></a>
#### confirmed

The field under validation must have a matching field of `foo_confirmation`. For example, if the field under validation is `password`, a matching `password_confirmation` field must be present in the input.

這個欄位必需與`foo_confirmation`欄位相同，舉例來說，假如驗證欄位是`password`，`password_confirmation`這個輸入欄位的值一樣

<a name="rule-date"></a>
#### date

The field under validation must be a valid date according to the `strtotime` PHP function.

這個欄位必需是合法的時間並且通過`strtotime`這個PHP函式

<a name="rule-date-format"></a>
#### date_format:_format_

The field under validation must match the _format_ defined according to the `date_parse_from_format` PHP function.
這個欄位的驗證必需與_format_定義的時間格式相同並且通過`date_parse_from_format`這個PHP函式

<a name="rule-different"></a>
#### different:_field_

The given _field_ must be different than the field under validation.
這個驗證必需與給予的欄位_field_不同

<a name="rule-digits"></a>
#### digits:_value_

The field under validation must be _numeric_ and must have an exact length of _value_.
這個欄位必需是數字而且長度要符合_value_

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

The field under validation must have a length between the given _min_ and _max_.
這個欄位的長度必需介於_min_ 與 _max_

<a name="rule-boolean"></a>
#### boolean

The field under validation must be able to be cast as a boolean. Accepted input are `true`, `false`, `1`, `0`, `"1"` and `"0"`.

這個欄位必需可以轉換成boolean(布林值true or false)，允許的值有`true`, `false`, `1`, `0`, `"1"` 及 `"0"`.

<a name="rule-email"></a>
#### email

The field under validation must be formatted as an e-mail address.
這個欄位必需符合email格式(xxxx@xxx.xxx)

<a name="rule-exists"></a>
#### exists:_table_,_column_

The field under validation must exist on a given database table.
這個欄位必需存在於資料庫的table中的欄位

#### Exists 規則的基本使用方法

	'state' => 'exists:states'

#### 指定一個自定的欄位名稱 Specifying A Custom Column Name

	'state' => 'exists:states,abbreviation'

You may also specify more conditions that will be added as "where" clauses to the query:
你也可以指定更多條件且那些條件將會新增至"where"查詢中

	'email' => 'exists:staff,email,account_id,1'
	(這個驗證規則的意思是 email必需符合staff這個表中email欄位且account_id=1)

Passing `NULL` as a "where" clause value will add a check for a `NULL` database value:
透過`NULL`搭配"where"的縮寫寫法去檢查資料庫的是否為`NULL`
	'email' => 'exists:staff,email,deleted_at,NULL'

<a name="rule-image"></a>
#### image

The file under validation must be an image (jpeg, png, bmp, or gif)
驗證檔案必需是個圖片(jpeg, png, bmp, or gif)

<a name="rule-in"></a>
#### in:_foo_,_bar_,...

The field under validation must be included in the given list of values.
這個欄位必需符合事先給予的清單的其中一個值

<a name="rule-integer"></a>
#### integer

The field under validation must have an integer value.
這個欄位必需是一個整數值

<a name="rule-ip"></a>
#### ip

The field under validation must be formatted as an IP address.
這個欄位必需符合IP位置的格式([1~255].[1~255].[1~255].[1~255])

<a name="rule-max"></a>
#### max:_value_

The field under validation must be less than or equal to a maximum _value_. Strings, numerics, and files are evaluated in the same fashion as the `size` rule.
這個欄位必需小於_value_，而字串，數字和檔案則是判斷`size`大小

<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

The file under validation must have a MIME type corresponding to one of the listed extensions.
這個檔案必需要有一個 MIME且必需對應清單中其中一個值

#### MIME規則基本用法

	'photo' => 'mimes:jpeg,bmp,png'

<a name="rule-min"></a>
#### min:_value_

The field under validation must have a minimum _value_. Strings, numerics, and files are evaluated in the same fashion as the `size` rule.
這個欄位必需大於_value_，而字串，數字和檔案則是判斷`size`大小

<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

The field under validation must not be included in the given list of values.
這個欄位的值必需不存在清單之中

<a name="rule-numeric"></a>
#### numeric

The field under validation must have a numeric value.
這個欄位必需是個數字(interger是指整數)

<a name="rule-regex"></a>
#### regex:_pattern_

The field under validation must match the given regular expression.
這個欄位必需符合你定義的正規表示法

**Note:** When using the `regex` pattern, it may be necessary to specify rules in an array instead of using pipe delimiters, especially if the regular expression contains a pipe character.

**注意:** 當使用`regex`模式時，你必需在陣列中指定一個正規表示法規則

<a name="rule-required"></a>
#### required

The field under validation must be present in the input data.
這個欄位必需要有值

<a name="rule-required-if"></a>
#### required\_if:_field_,_value_

The field under validation must be present if the _field_ field is equal to _value_.
這個欄位必需符合_field_等於_value_的條件

<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

The field under validation must be present _only if_ any of the other specified fields are present.

<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

The field under validation must be present _only if_ all of the other specified fields are present.

<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

The field under validation must be present _only when_ any of the other specified fields are not present.

<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

The field under validation must be present _only when_ the all of the other specified fields are not present.

<a name="rule-same"></a>
#### same:_field_

The given _field_ must match the field under validation.


<a name="rule-size"></a>
#### size:_value_

The field under validation must have a size matching the given _value_. For string data, _value_ corresponds to the number of characters. For numeric data, _value_ corresponds to a given integer value. For files, _size_ corresponds to the file size in kilobytes.

<a name="rule-timezone"></a>
#### timezone

The field under validation must be a valid timezone identifier according to the `timezone_identifiers_list` PHP function.
這個欄位必需是個正確的時區並且通過`date_parse_from_format`這個PHP函式

<a name="rule-unique"></a>
#### unique:_table_,_column_,_except_,_idColumn_

The field under validation must be unique on a given database table. If the `column` option is not specified, the field name will be used.
這個欄位必需是在資料表中唯一的值，假如 `column` 沒有指定就會用欄位名稱來替代 `column` 

#### Unique 規則的基本用法

	'email' => 'unique:users'

#### 指定一個自訂的欄位名稱

	'email' => 'unique:users,email_address'

#### 強制一個Unique規則去忽略一個ID

	'email' => 'unique:users,email_address,10'

#### 新增一個額外的 Where語法

你也可以指定更多條件而這些條件將會新增到查詢的Where語法

	'email' => 'unique:users,email_address,NULL,id,account_id,1'

以上的規則是在說明 只有`account_id`為`1`的列會被加入唯一值的檢驗

<a name="rule-url"></a>
#### url

這個欄位必需符合URL的格式驗證

> **註記:** 這個函式使用PHP的`filter_var`方法

<a name="conditionally-adding-rules"></a>
## 有條件的新增規則

在某些狀況，假如這個欄位是輸入一個陣列，你會希望針對特定欄位執行驗證，為了快速完成這個驗證，你可以在你的規則語法中使用`sometimes`參數
In some situations, you may wish to run validation checks against a field **only** if that field is present in the input array. To quickly accomplish this, add the `sometimes` rule to your rule list:

	$v = Validator::make($data, array(
		'email' => 'sometimes|required|email',
	));

在以上的例子中，將只會驗證`$data`陣列中的`email`欄位

#### 複雜的條件驗證

Sometimes you may wish to require a given field only if another field has a greater value than 100. Or you may need two fields to have a given value only when another field is present. Adding these validation rules doesn't have to be a pain. First, create a `Validator` instance with your _static rules_ that never change:
有時你會希望驗證一個必要的欄位，只有在另一個欄位大於100的時後，或是你也會需要一個欄位有一個值當另外的欄位也有值的時後，新增這些驗證規則並不會很痛苦，首先新增一個`Validator`實例並且使用_static rules_且不要改變：

	$v = Validator::make($data, array(
		'email' => 'required|email',
		'games' => 'required|numeric',
	));

Let's assume our web application is for game collectors. If a game collector registers with our application and they own more than 100 games, we want them to explain why they own so many games. For example, perhaps they run a game re-sell shop, or maybe they just enjoy collecting. To conditionally add this requirement, we can use the `sometimes` method on the `Validator` instance.
讓我們假設我們的網頁程式是一個遊戲收納器，當一個遊戲收納器的註冊者使用我們的應用程式且他們擁有大於100個遊戲，我們希望他們解釋為什麼他們擁有這麼多遊戲，舉列來說他們也許執行一個遊戲二手商店或是他們可能只是喜歡收藏，有條件的新增這些需求，我們可以使用`sometimes`這個方法。

	$v->sometimes('reason', 'required|max:500', function($input)
	{
		return $input->games >= 100;
	});

The first argument passed to the `sometimes` method is the name of the field we are conditionally validating. The second argument is the rules we want to add. If the `Closure` passed as the third argument returns `true`, the rules will be added. This method makes it a breeze to build complex conditional validations. You may even add conditional validations for several fields at once:
`sometimes`這個方法的第一個參數是欄位的名稱，說明哪些欄位需要有條件式的驗證，第二個參數是我們想增加的規則，假如`Closure`也就是第三個參數回傳`true`，這個額外的規則就會變新增，這個方法讓我們輕易的建立一個複雜且有條件式的驗證，你甚至可以一次為多個欄位新增條件式驗證：

	$v->sometimes(array('reason', 'cost'), 'required', function($input)
	{
		return $input->games >= 100;
	});

> **註記:** `$input`參數傳遞到你的`Closure`將會是一個`Illuminate\Support\Fluent`實例且他會是一個物件來存取你的輸入欄位

<a name="custom-error-messages"></a>
## 自定錯誤訊息Custom Error Messages

如果有需要你也許會在一個驗證中使用預設的自定錯誤訊息，這裡有幾個方法去指定自定訊息

#### 透過驗證器傳遞自定參數

	$messages = array(
		'required' => 'The :attribute field is required.',
	);

	$validator = Validator::make($input, $rules, $messages);

> *註記:* `:attribute`我們稱為place-holders將會在驗證時用欄位的名稱取代，你也可以在驗證訊息中使用其他的place-holders


#### 其他驗證 Place-Holders

	$messages = array(
		'same'    => 'The :attribute and :other must match.',
		'size'    => 'The :attribute must be exactly :size.',
		'between' => 'The :attribute must be between :min - :max.',
		'in'      => 'The :attribute must be one of the following types: :values',
	);

#### 指定一個自定訊息給一個屬性

你也許希望去指定一個自定錯誤訊息針對特定的欄位：

	$messages = array(
		'email.required' => 'We need to know your e-mail address!',
	);

<a name="localization"></a>
#### 多語系檔案中宣告自定訊息

In some cases, you may wish to specify your custom messages in a language file instead of passing them directly to the `Validator`. To do so, add your messages to `custom` array in the `app/lang/xx/validation.php` language file.
在有些案例中，你也許希望指定你的自定訊息在一個語系檔直接的替代`Validator`中的自定訊息，所以請新增你的訊息到`app/lang/xx/validation.php`語系統的`custom`陣列中

	'custom' => array(
		'email' => array(
			'required' => 'We need to know your e-mail address!',
		),
	),

<a name="custom-validation-rules"></a>
## 自定驗證規則

#### 註冊一個自定驗證規則

Laravel provides a variety of helpful validation rules; however, you may wish to specify some of your own. One method of registering custom validation rules is using the `Validator::extend` method:

Laravel提供一個方法幫助你改變驗證規則;然而你也許希望指定一些屬於你自已的驗證規則，這個時後可以使用`Validator::extend`註冊一個自定的驗證規則：

	Validator::extend('foo', function($attribute, $value, $parameters)
	{
		return $value == 'foo';
	});

The custom validator Closure receives three arguments: the name of the `$attribute` being validated, the `$value` of the attribute, and an array of `$parameters` passed to the rule.
自定的驗證Closure接收三個參數：

`$attribute`->驗證規則的名稱
`$value`->屬性
`$parameters`->一個陣列傳到驗證規則

你也可以透過一個類別的方法來替代`extend` 的Closure：

	Validator::extend('foo', 'FooValidator@validate');

記註你也將會需要為你的自定規則定義一個錯誤訊息，你可以這麼做或直接寫一個自定訊息陣列或新增一個驗證的語系檔
Note that you will also need to define an error message for your custom rules. You can do so either using an inline custom message array or by adding an entry in the validation language file.

#### 擴充Validator Class

Instead of using Closure callbacks to extend the Validator, you may also extend the Validator class itself. To do so, write a Validator class that extends `Illuminate\Validation\Validator`. You may add validation methods to the class by prefixing them with `validate`:

使用一個Closure繼承Validator來替代原本的Validator，你也可以繼承Validator類別，在你的類別中繼承`Illuminate\Validation\Validator`. 你就可以新增驗證方法在類別中
	<?php

	class CustomValidator extends Illuminate\Validation\Validator {

		public function validateFoo($attribute, $value, $parameters)
		{
			return $value == 'foo';
		}

	}

#### Registering A Custom Validator Resolver

Next, you need to register your custom Validator extension:

	Validator::resolver(function($translator, $data, $rules, $messages)
	{
		return new CustomValidator($translator, $data, $rules, $messages);
	});

When creating a custom validation rule, you may sometimes need to define custom place-holder replacements for error messages. You may do so by creating a custom Validator as described above, and adding a `replaceXXX` function to the validator.

	protected function replaceFoo($message, $attribute, $rule, $parameters)
	{
		return str_replace(':foo', $parameters[0], $message);
	}

If you would like to add a custom message "replacer" without extending the `Validator` class, you may use the `Validator::replacer` method:

	Validator::replacer('rule', function($message, $attribute, $rule, $parameters)
	{
		//
	});
=======
# 驗證

- [基本用法](#basic-usage)
- [使用錯誤訊息](#working-with-error-messages)
- [錯誤訊息 & 視圖](#error-messages-and-views)
- [使用驗證規則](#available-validation-rules)
- [有條件新增規則](#conditionally-adding-rules)
- [自訂錯誤訊息](#custom-error-messages)
- [自訂驗證規則](#custom-validation-rules)

<a name="basic-usage"></a>
## 基本用法

Laravel 透過 `Validation` 類別讓你可以簡單、方便的驗證資料正確性及查看驗證的錯誤訊息。

#### 基本驗證範例

	$validator = Validator::make(
		array('name' => 'Dayle'),
		array('name' => 'required|min:5')
	);


我們透過 `make` 這個方法來的第一個參數設定該被驗證資料名稱，然後第二個參數我們定義該資料可被接受的規則。

#### 使用陣列來定義規則

多個規則可以使用"|"符號分隔，或是單一陣列作為單獨的元素分隔。

	$validator = Validator::make(
		array('name' => 'Dayle'),
		array('name' => array('required', 'min:5'))
	);

#### 驗證多個欄位

    $validator = Validator::make(
        array(
            'name' => 'Dayle',
            'password' => 'lamepassword',
            'email' => 'email@example.com'
        ),
        array(
            'name' => 'required',
            'password' => 'required|min:8',
            'email' => 'required|email|unique:users'
        )
    );


當一個 `Validator` 實例被建立，`fails`（或 `passes`） 這二個方法就可以在驗證時使用，如下：

	if ($validator->fails())
	{
		// The given data did not pass validation
	}


假如驗證失敗，您可以從驗證器中接收錯誤資訊。

	$messages = $validator->messages();


您可能不需要錯誤訊息，只想取得無法通過驗證的規則，您可以使用 'failed' 方法：

	$failed = $validator->failed();

#### 驗證檔案

`Validator` 類別提供了一些規則用來驗證檔案，例如 `size`, `mimes` 等等。當需要驗證檔案時，你僅需將它們和你其他的資料一同送給驗證器即可。

<a name="working-with-error-messages"></a>
## 使用錯誤訊息

當你呼叫一個 `Validator` 實例的 `messages` 方法後，你會得到一個 `MessageBag` 的變數，裡面有許多方便的方法讓你取得錯誤訊息。

#### 查看一個欄位的第一個錯誤訊息

	echo $messages->first('email');

#### 查看一個欄位的所有錯誤訊息

	foreach ($messages->get('email') as $message)
	{
		//
	}

#### 查看所有欄位的所有錯誤訊息

	foreach ($messages->all() as $message)
	{
		//
	}

#### 判斷一個欄位是否有錯誤訊息

	if ($messages->has('email'))
	{
		//
	}

#### 錯誤訊息格式化輸出

	echo $messages->first('email', '<p>:message</p>');

> **注意:** 預設錯誤訊息以 Bootstrap 相容語法輸出。

#### 查看所有錯誤訊息並以格式化輸出

	foreach ($messages->all('<li>:message</li>') as $message)
	{
		//
	}

<a name="error-messages-and-views"></a>
## 錯誤訊息 & 視圖

當你開始進行驗證，你將會需要一個簡易的方法去取得錯誤訊息並回傳到你的視圖中，在 Laravel 你可以很方便的處理這些事，你可以透過下面的路由例子來了解：


	Route::get('register', function()
	{
		return View::make('user.register');
	});

	Route::post('register', function()
	{
		$rules = array(...);

		$validator = Validator::make(Input::all(), $rules);

		if ($validator->fails())
		{
			return Redirect::to('register')->withErrors($validator);
		}
	});

記得當驗證失敗後，我們會使用 `withErrors` 方法來將 `Validator` 實例進行重新導向。這方法會將錯誤訊息存入 session 中，這樣才能在下個請求中被使用。

然而，我們並不需要特別去將錯誤訊息綁定在我們 GET 路由的視圖中。因為 Laravel 會確認在 Session 資料中是否有錯誤訊息，並且自動將它們綁定至視圖中。**所以請注意，`$errors` 變數存在於所有的視圖中，所有的請求裡，**讓你可以直接假設 `$errors` 變數已被定義且可以安全地使用。`$errors` 變數是 `MessageBag` 類別的一個實例。

所以，重新導向之後，你可以自然的在視圖中使用 `$errors` 變數：

	<?php echo $errors->first('email'); ?>

### 命名錯誤清單

假如你在一個頁面中有許多的表單, 你可能希望為錯誤命名一個`MessageBag`. 這將讓你針對特定的表單查看其錯誤訊息, 我們只要簡單的在 `withErrors` 的第二個參數設定名稱即可：

	return Redirect::to('register')->withErrors($validator, 'login');

接著你可以從一個 `$errors` 變數中取得已命名的 `MessageBag` 實例：

	<?php echo $errors->login->first('email'); ?>

<a name="available-validation-rules"></a>
## 既存的驗證規則

以下是既存的驗證規則清單與他們的函式名稱：

- [Accepted](#rule-accepted)
- [Active URL](#rule-active-url)
- [After (Date)](#rule-after)
- [Alpha](#rule-alpha)
- [Alpha Dash](#rule-alpha-dash)
- [Alpha Numeric](#rule-alpha-num)
- [Array](#rule-array)
- [Before (Date)](#rule-before)
- [Between](#rule-between)
- [Boolean](#rule-boolean)
- [Confirmed](#rule-confirmed)
- [Date](#rule-date)
- [Date Format](#rule-date-format)
- [Different](#rule-different)
- [Digits](#rule-digits)
- [Digits Between](#rule-digits-between)
- [E-Mail](#rule-email)
- [Exists (Database)](#rule-exists)
- [Image (File)](#rule-image)
- [In](#rule-in)
- [Integer](#rule-integer)
- [IP Address](#rule-ip)
- [Max](#rule-max)
- [MIME Types](#rule-mimes)
- [Min](#rule-min)
- [Not In](#rule-not-in)
- [Numeric](#rule-numeric)
- [Regular Expression](#rule-regex)
- [Required](#rule-required)
- [Required If](#rule-required-if)
- [Required With](#rule-required-with)
- [Required With All](#rule-required-with-all)
- [Required Without](#rule-required-without)
- [Required Without All](#rule-required-without-all)
- [Same](#rule-same)
- [Size](#rule-size)
- [Timezone](#rule-timezone)
- [Unique (Database)](#rule-unique)
- [URL](#rule-url)

<a name="rule-accepted"></a>
#### accepted

欄位值為 _yes_, _on_, 或是 _1_ 時，驗證才會通過。這在確認"服務條款"是否同意時很有用。

<a name="rule-active-url"></a>
#### active_url

欄位值透過 PHP 函式 `checkdnsrr` 來驗證是否為一個有效的網址。

<a name="rule-after"></a>
#### after:_date_

驗證欄位是否是在指定日期之後。這個日期將會使用 PHP `strtotime` 函式驗證。

<a name="rule-alpha"></a>
#### alpha

欄位僅全數為字母字元時通過驗證。

<a name="rule-alpha-dash"></a>
#### alpha_dash

欄位值僅允許字母、數字、破折號（-）以及底線（_）

<a name="rule-alpha-num"></a>
#### alpha_num

欄位值僅允許字母、數字

<a name="rule-array"></a>
#### array

欄位值僅允許為陣列

<a name="rule-before"></a>
#### before:_date_

驗證欄位是否是在指定日期之前。這個日期將會使用 PHP `strtotime` 函式驗證。

<a name="rule-between"></a>
#### between:_min_,_max_

欄位值需介於指定的 _min_ 和 _max_ 值之間。字串、數值或是檔案都是用同樣的方式來進行驗證。


<a name="rule-confirmed"></a>
#### confirmed

欄位值需與對應的欄位值 `foo_confirmation` 相同。例如，如果驗證的欄位是 `password` ，那對應的欄位 `password_confirmation` 就必須存在且與 `password` 欄位相符。

<a name="rule-date"></a>
#### date

欄位值透過 PHP `strtotime` 函式驗證是否為一個合法的日期。

<a name="rule-date-format"></a>
#### date_format:_format_

欄位值透過 PHP `date_parse_from_format` 函式驗證符合 _format_ 制定格式的日期是否為合法日期。

<a name="rule-different"></a>
#### different:_field_

欄位值需與指定的欄位 _field_ 值不同。


<a name="rule-digits"></a>
#### digits:_value_

欄位值需為數字且長度需為 _value_。

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

欄位值需為數字，且長度需介於 _min_ 與 _max_ 之間。

<a name="rule-boolean"></a>
#### boolean

欄位必須可以轉換成布林值，可接受的值為 `true`, `false`, `1`, `0`, `"1"`, `"0"`。

<a name="rule-email"></a>
#### email

欄位值需符合 email 格式。

<a name="rule-exists"></a>
#### exists:_table_,_column_

欄位值需與存在於資料庫 _table_ 中的 _column_ 欄位值其一相同。

#### Exists 規則的基本使用方法

	'state' => 'exists:states'

#### 指定一個自訂的欄位名稱

	'state' => 'exists:states,abbreviation'

你可以指定更多條件且那些條件將會被新增至 "where" 查詢裡：

	'email' => 'exists:staff,email,account_id,1'
	/* 這個驗證規則為 email 需存在於 staff 這個資料表中 email 欄位中且 account_id=1 */

透過`NULL`搭配"where"的縮寫寫法去檢查資料庫的是否為`NULL`

	'email' => 'exists:staff,email,deleted_at,NULL'

<a name="rule-image"></a>
#### image

檔案必需為圖片(jpeg, png, bmp 或 gif)

<a name="rule-in"></a>
#### in:_foo_,_bar_,...

欄位值需符合事先給予的清單的其中一個值

<a name="rule-integer"></a>
#### integer

欄位值需為一個整數值

<a name="rule-ip"></a>
#### ip

欄位值需符合 IP 位址格式。

<a name="rule-max"></a>
#### max:_value_

欄位值需小於等於 _value_。字串、數字和檔案則是判斷 `size` 大小。

<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

檔案的 MIME 類別需在給定清單中的列表中才能通過驗證。

#### MIME規則基本用法

	'photo' => 'mimes:jpeg,bmp,png'

<a name="rule-min"></a>
#### min:_value_

欄位值需大於等於 _value_。字串、數字和檔案則是判斷 `size` 大小。

<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

欄位值不得為給定清單中其一。

<a name="rule-numeric"></a>
#### numeric

欄位值需為數字。

<a name="rule-regex"></a>
#### regex:_pattern_

欄位值需符合給定的正規表示式。

**注意:** 當使用`regex`模式時，你必須使用陣列來取代"|"作為分隔，尤其是當正規表示式中含有"|"字元。

<a name="rule-required"></a>
#### required

欄位值為必填。

<a name="rule-required-if"></a>
#### required\_if:_field_,_value_

欄位值在 _field_ 欄位值為 _value_ 時為必填。

<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

欄位值 _僅在_ 任一指定欄位有值情況下為必填。

<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

欄位值 _僅在_ 所有指定欄位皆有值情況下為必填。

<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

欄位值 _僅在_ 任一指定欄位沒有值情況下為必填。

<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

欄位值 _僅在_ 所有指定欄位皆沒有值情況下為必填。

<a name="rule-same"></a>
#### same:_field_

欄位值需與指定欄位 _field_ 等值。

<a name="rule-size"></a>
#### size:_value_

欄位值的尺寸需符合給定 _value_ 值。對於字串來說，_value_ 為需符合的字元長度。對於數字來說，_value_ 為需符合的整數值。對於檔案來說，_value_ 為需符合的檔案大小（單位 kb)。

<a name="rule-timezone"></a>
#### timezone

欄位值透過 PHP `timezone_identifiers_list` 函式來驗證是否為有效的時區。

<a name="rule-unique"></a>
#### unique:_table_,_column_,_except_,_idColumn_

欄位值在給定的資料庫中需為唯一值。如果 `column（欄位）` 選項沒有指定，將會使用欄位名稱。

#### 唯一(Unique)規則的基本用法

	'email' => 'unique:users'

#### 指定一個自訂的欄位名稱

	'email' => 'unique:users,email_address'

#### 強制唯一規則忽略指定的 ID

	'email' => 'unique:users,email_address,10'

#### 增加額外的 Where 條件

你也可以指定更多的條件式到 "where" 查詢語句中：

	'email' => 'unique:users,email_address,NULL,id,account_id,1'

上述規則為只有 `account_id` 為 `1` 的資料列會做唯一規則的驗證。

<a name="rule-url"></a>
#### url

欄位值需符合 URL 的格式。

> **注意:** 此函式會使用 PHP `filter_var` 方法驗證。

<a name="conditionally-adding-rules"></a>
## 有條件新增規則

某些情況下，你可能 **只想** 當欄位有值時，才進行驗證。只要增加 `sometimes` 條件進條件列表中，就可以快速達成：

	$v = Validator::make($data, array(
		'email' => 'sometimes|required|email',
	));

在上述範例中，`email` 欄位只會在當其在 `$data` 陣列中有值的情況下才會被驗證。

#### 複雜的條件式驗證

有時，你可以希望給定欄位在其他欄位有超過 100 時為必填。或者你希望兩個欄位，當其一欄位有值時，另一欄位將會有一個給定的值。增加這樣的驗證條件並不痛苦。首先，利用你尚未更動的 _靜態規則_ 創建一個 `Validator` 實例：

	$v = Validator::make($data, array(
		'email' => 'required|email',
		'games' => 'required|numeric',
	));

假設我們的網頁應用程式是專為遊戲收藏家所設計。如果遊戲收藏家收藏超過一百款遊戲，我們希望他們說明為什麼他們擁有這麼多遊戲。像是，可能他們經營一家二手遊戲商店，或是他們可能只是享受收集的樂趣。有條件的加入此需求，我們可以在 `Validator` 實例中使用 `sometimes` 方法。

	$v->sometimes('reason', 'required|max:500', function($input)
	{
		return $input->games >= 100;
	});

傳遞至 `sometimes` 方法的第一個參數是我們要條件式認證的欄位名稱。第二個參數是我們想加入驗證規則。 `閉包（Closure）` 作為第三個參數傳入，如果回傳值為 `true` 那該規則就會被加入。這個方法可以輕而易舉的建立複雜的條件式驗證。你也可以一次對多個欄位增加條件式驗證：

	$v->sometimes(array('reason', 'cost'), 'required', function($input)
	{
		return $input->games >= 100;
	});

> **注意:** 傳遞至你的 `Closure` 的 `$input` 參數為 `Illuminate\Support\Fluent` 的實例且用來作為存取你的輸入及檔案的物件。

<a name="custom-error-messages"></a>
## 自訂錯誤訊息

如果需要，你可以為驗證自訂錯誤訊息取代預設錯誤訊息。這裏有幾個方式可以設定客制訊息。

#### 傳遞客制訊息進驗證器

	$messages = array(
		'required' => 'The :attribute field is required.',
	);

	$validator = Validator::make($input, $rules, $messages);

> **注意:** 在驗證中，`:attribute` 佔位符會被欄位的實際名稱給取代。你也可以在驗證訊息中使用其他的佔位符。

#### 其他的驗證佔位符

	$messages = array(
		'same'    => 'The :attribute and :other must match.',
		'size'    => 'The :attribute must be exactly :size.',
		'between' => 'The :attribute must be between :min - :max.',
		'in'      => 'The :attribute must be one of the following types: :values',
	);

#### 為特定屬性給予一個客制化訊息

有時你只想為一個特定欄位指定一個客制錯誤訊息：

	$messages = array(
		'email.required' => 'We need to know your e-mail address!',
	);

<a name="localization"></a>
#### 在語言檔中指定客制訊息

某些狀況下，你可能希望在語言檔中設定你的客制訊息，而非直接將他們傳遞給 `Validator`。要達到目的，將你的訊息增加至 `app/lang/xx/validation.php` 檔案的 `custom` 陣列中。

	'custom' => array(
		'email' => array(
			'required' => 'We need to know your e-mail address!',
		),
	),

<a name="custom-validation-rules"></a>
## 自訂驗證規則

#### 註冊自訂驗證規則

Laravel 提供了各種有用的驗證規則，但是，你可能希望可以設定一些自己專用的。註冊自訂的驗證規則的方法之一就是使用 `Validator::extend` 方法：

	Validator::extend('foo', function($attribute, $value, $parameters)
	{
		return $value == 'foo';
	});

客制驗證器閉包接收三個參數：要被驗證的 `$attribute(屬性)` 的名稱，屬性的值 `$value`，傳遞至驗證規則的 `$parameters` 陣列。

你同樣可以傳遞一個類別和方法到 `extend` 方法中，取代原本的閉包：

	Validator::extend('foo', 'FooValidator@validate');

注意,你同時需要為你的自訂規則訂立一個錯誤訊息。你可以使用行內自訂訊息陣列或是在認證語言檔裡新增。

#### 擴展 Validator 類別

除了使用閉包回呼(Closure callbacks)來擴展 Validator 外，你一樣可以直接擴展 Validator 類別。你可以寫一個擴展自 `Illuminate\Validation\Validator` 的驗證器類別。你也可以增加驗證方法到以 `validate` 為開頭的類別中：


	<?php

	class CustomValidator extends Illuminate\Validation\Validator {

		public function validateFoo($attribute, $value, $parameters)
		{
			return $value == 'foo';
		}

	}

#### Registering A Custom Validator Resolver

接下來，你需要註冊你自訂驗證器擴展：

	Validator::resolver(function($translator, $data, $rules, $messages)
	{
		return new CustomValidator($translator, $data, $rules, $messages);
	});

當創建自訂驗證規則時，你可能有時需要為錯誤訊息定義客制化的佔位符。你可以如上所述創建一個自訂的驗證器，然後增加 `replaceXXX` 函式進驗證器中。

	protected function replaceFoo($message, $attribute, $rule, $parameters)
	{
		return str_replace(':foo', $parameters[0], $message);
	}

如果你想要增加一個自訂訊息 "replacer" 但不擴展 `Validator` 類別，你可以使用 `Validator::replacer` 方法：

	Validator::replacer('rule', function($message, $attribute, $rule, $parameters)
	{
		//
	});
>>>>>>> dedcbfab3e00b3f206112d3a1174623c3d721d8f
