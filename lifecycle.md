# 請求的生命週期

- [介紹](#introduction)
- [生命週期的概述](#lifecycle-overview)
- [把焦點放到服務提供者上](#focus-on-service-providers)

<a name="introduction"></a>
## 介紹

在「真實世界」中使用任何工具時，如果能夠理解工具運作的原理，會更加有自信心。開發應用程式這件事也不例外。當你理解開發工具的功能時，使用起來會更加得心應手。

本文件的目的在於為你提供更進階的 Laravel 框架運作原理。當你愈是瞭解整體框架，就越能降低「莫名其妙」的感覺，並且會讓你更有信心建構應用程式。如果你不明白所有的專業術語，別太難過！只要嘗試對眼前不懂的地方有個基本的理解，你的知識會隨著你瀏覽文件的其他章節時跟著成長。

<a name="lifecycle-overview"></a>
## 生命週期的概述

### 第一件事

`public/index.php` 這個檔案是對 Laravel 應用程式所有請求的進入點。所有請求都會透過你的網頁伺服器（Apache 或 Nginx）設定。`index.php` 檔案並沒有放入太多程式碼。更確切地說，它只是一個載入框架其他部分的起點。

`index.php` 檔案會去載入 Composer 產生的自動載入的定義，並接收一個從 `bootstrap/app.php` 腳本中的 Laravel 應用程式實例。Laravel 本身的第一個動作就是建立一個應用程式 / [服務容器](/docs/{{version}}/container)的實例

### HTTP / 終端核心

Next, the incoming request is sent to either the HTTP kernel or the console kernel, depending on the type of request that is entering the application. These two kernels serve as the central location that all requests flow through. For now, let's just focus on the HTTP kernel, which is located in `app/Http/Kernel.php`.

The HTTP kernel extends the `Illuminate\Foundation\Http\Kernel` class, which defines an array of `bootstrappers` that will be run before the request is executed. These bootstrappers configure error handling, configure logging, [detect the application environment](/docs/{{version}}/configuration#environment-configuration), and perform other tasks that need to be done before the request is actually handled.

The HTTP kernel also defines a list of HTTP [middleware](/docs/{{version}}/middleware) that all requests must pass through before being handled by the application. These middleware handle reading and writing the [HTTP session](/docs/{{version}}/session), determining if the application is in maintenance mode, [verifying the CSRF token](/docs/{{version}}/csrf), and more.

The method signature for the HTTP kernel's `handle` method is quite simple: receive a `Request` and return a `Response`. Think of the Kernel as being a big black box that represents your entire application. Feed it HTTP requests and it will return HTTP responses.

#### 服務提供者

One of the most important Kernel bootstrapping actions is loading the [service providers](/docs/{{version}}/providers) for your application. All of the service providers for the application are configured in the `config/app.php` configuration file's `providers` array. First, the `register` method will be called on all providers, then, once all providers have been registered, the `boot` method will be called.

Service providers are responsible for bootstrapping all of the framework's various components, such as the database, queue, validation, and routing components. Since they bootstrap and configure every feature offered by the framework, service providers are the most important aspect of the entire Laravel bootstrap process.

#### 指派請求

Once the application has been bootstrapped and all service providers have been registered, the `Request` will be handed off to the router for dispatching. The router will dispatch the request to a route or controller, as well as run any route specific middleware.

<a name="focus-on-service-providers"></a>
## 把焦點放到服務提供者上

Service providers are truly the key to bootstrapping a Laravel application. The application instance is created, the service providers are registered, and the request is handed to the bootstrapped application. It's really that simple!

Having a firm grasp of how a Laravel application is built and bootstrapped via service providers is very valuable. Of course, your application's default service providers are stored in the `app/Providers` directory.

By default, the `AppServiceProvider` is fairly empty. This provider is a great place to add your application's own bootstrapping and service container bindings. Of course, for large applications, you may wish to create several service providers, each with a more granular type of bootstrapping.
