# 用 Laravel 建立一個 SAML2 的 SSO 網站群

## SAML 的基本觀念請參考 [SAML for Web Developers](https://github.com/jch/saml)

## 使用套件

### SP [Saml2 Service Provider](https://github.com/SocialiteProviders/Saml2)

### IDP [Laravel SAML IdP](https://github.com/codegreencreative/laravel-samlidp)

## 架構

已先建立三個 Laravel 11 的網站，分別是：

- `auth.jk-web.com` IDP
- `shop.jk-web.com` SP
- `member.jk-web.com` SP

## 步驟

### SP

在 sp 上安裝 [Saml2 Service Provider](https://github.com/SocialiteProviders/Saml2)

```bash
composer require socialiteproviders/saml2
```

修改 Laravel 相關檔案內容

### `app/Providers/AppServiceProvider.php`

```php
use Illuminate\Support\Facades\Event;

...

  public function boot(): void
  {
    // 加入事件監聽
    Event::listen(function (\SocialiteProviders\Manager\SocialiteWasCalled $event) {
        $event->extendSocialite('saml2', \SocialiteProviders\Saml2\Provider::class);
    });
  }

```

### `bootstrap/providers.php`

```php
return [
  App\Providers\AppServiceProvider::class,
  SocialiteProviders\Manager\ServiceProvider::class,  // 加入此行
];
```

### 建立 cert 及 key 檔

```bash
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout sp_key.pem -out sp_cert.pem
```

我是將 certifications 放在 storage/saml2 目錄下，下面 `config/services.php` 中的設定也對應此處。

產生的憑 cert 檔需要在 IDP 的 config/samlidp.php 檔案中，針對對應的 SP 進行指定。 可以將憑證檔案直接存放在 IDP 伺服器上，或將憑證的內容以字串形式嵌入到設定檔中。

### `config/services.php`

```php
// 加入 SAML 的設定
'saml2' => [
  'metadata' => env('SAML2_META_URL'),
  // IDP 的 metadata URL, 我將 IDP metadata URL 定義在 .env 檔中，以便區隔測試環境及正式環境
  'sp_certificate' => file_get_contents(storage_path('saml2/sp_cert.pem')), // 之前產生的憑證檔案
  'sp_private_key' => file_get_contents(storage_path('saml2/sp_key.pem')),
  'sp_sign_assertions' => true, // 這邊設為 true 及  false 沒什麼差別
  'sp_sls' => 'saml/logout',  // IDP 登出時會訪問這個 route
],
```

### `routes/web.php`

```php
/**
 * SSO URL
 *
 * 當使用者需要進行登入驗證時，將頁面轉到這個 route，Socialite 會產生 SAMLRequest 所需資料並轉址到 IDP，
 * 如果使用者在 IDP 未曾登入，會停留在 IDP 的登入頁面，登入成功之後會再轉回 SP 的 ACS URL。
 * 如果使用者在 IDP 已是登入狀態，則直接轉回 SP 的 ACS。
 *
 */
Route::get('/auth/redirect', function () {
  return Socialite::driver('saml2')->redirect();
});

/**
 * ACS URL
 *
 * SP 的 ACS URL，IDP 驗證完成之後，會將使用者資料回傳至這個 route，在此可再進行 SP 方的登入驗證處理。
 *
 */
Route::post('/auth/callback', function () {
    $user = Socialite::driver('saml2')->user();
    $user = User::where('email', $user->email)->first();
    Auth::login($user);
    return redirect('/');
});

/**
 * 發起 SLO
 *
 * 對 IDP 發送 SLO 請求用
 *
 */
Route::get('/saml/logout', function () {
    return redirect(env('SAML2_LOGOUT_URL'));
});

/**
 * SLO URL
 *
 * 這裡是供 IDP 訪問用，
 * 在收到任一 SP 發出的 SLO 請求後，IDP 會依序存取所有已建立 SSO SESSION 的 SP 的 SSO URL。
 * 此處可進行 SP 端的 Logout 相關動作
 * 最後一定要 return $response 才會接力轉址到下一個 SP，完成完整的 SLO 流程。
 */
Route::get('/auth/logout', function () {
    $response = Socialite::driver('saml2')->logoutResponse();
    Auth::logout();
    return $response;
});

```

### `.env`

以 shop.jk-web.com 為例

```bash
SAML2_META_URL=https://auth.jk-web.com/saml/metadata
SAML2_LOGOUT_URL=https://auth.jk-web.com/saml/logout?return_to=shop.jk-web.com
# 在 IDP 的 SLO route 需加上 return_to 參數，讓瀏覽器頁面在輪流訪問完所有已建立 SSO session 的 SP 之後，回到發起 SLO 的 SP，此處的參數值需要與 IDP samlidp config 檔中的 sp_slo_redirects 設定對應
```

### `bootstrap/app.php`

因為 ACS URL 是 POST 方法，需要取消 CSRF 驗證，才能讓 IDP 成功訪問。

```php
->withMiddleware(function (Middleware $middleware) {
  $middleware->validateCsrfTokens(except: [
      'auth/callback',
  ]);
})
```

## IDP

在 IDP 安裝 [Laravel SAML IdP](https://github.com/codegreencreative/laravel-samlidp)

```php
composer require codegreencreative/laravel-samlidp
```

Publish 設定檔

```bash
php artisan vendor:publish --tag="samlidp_config"
```

修改相關檔案

### `config/filesystem.php`

```php
'disks' => [
  ...

  'samlidp' => [
      'driver' => 'local',
      'root' => storage_path() . '/samlidp',
  ]
],
```

使用 `php artisan samlidp:cert` 指令產生憑證檔案時，會存放在這裡設定的位置。

### 產生憑證檔案

有別於 SP 需要自行輸入 openssl 指令，[Laravel SAML IdP](https://github.com/codegreencreative/laravel-samlidp) 為我們建立了 `php artisan` 指定來做這件事情。

```bash
php artisan samlidp:cert
```

### 設定 SP

#### `config/samlidp.php`

主要是設定 `sp` 及 `sp_slo_redirects` 這兩項資料, 其它都用預設值。因為要區隔測試環境及正式環境，因為這兩項設定值都移到另一個 config 檔 `config/samlhosts.php` 中，並且設定 .gitignore 將其排除在 git 追蹤的檔案之外。

```php
'sp' => config('samlhosts.sps'),
'sp_slo_redirects' => config('samlhosts.slo_redirects'),
```

### .gitignore

```
/config/samlhosts.php
```

#### `config/samlhosts.php`

```php
<?php
return [
  "sps" => [
    'aHR0cHM6Ly9zaG9wLmprLXdlYi5jb20vYXV0aC9jYWxsYmFjaw==' => [
      'destination' => 'https://shop.jk-web.com/auth/callback',
      'logout' => 'https://shop.jk-web.com/auth/logout',
      'certificate' => 'file://' . storage_path('samlidp/shop.cert'),
    ],
    'aHR0cHM6Ly9tZW1iZXIuamstd2ViLmNvbS9hdXRoL2NhbGxiYWNr' => [
      'destination' => 'https://member.jk-web.com/auth/callback',
      'logout' => 'https://member.jk-web.com/auth/logout',
      'certificate' => 'file://' . storage_path('samlidp/member.cert'),
    ]
  ],
  "slo_redirects" => [
    'shop.jk-web.com' => 'https://shop.jk-web.com',
    'member.jk-web.com' => 'https://member.jk-web.com',
  ],
];
```

`sps` 下的每一個陣列元素就是一個 SP 的設定。

以 shop.jk-web.com 為例，其鍵值 `aHR0cHM6Ly9zaG9wLmprLXdlYi5jb20vYXV0aC9jYWxsYmFjaw==` 就是 這個 SP 的 ACS URL 的 base64 encode 字串。
`destination` 就是 SP ACS URL，`logout` 就是 SP SLO URL。`certificate` 就是 SP 當初產生的憑證檔中 cert 的部份。

`slo_redirects` 的每一筆資料就是每個 SP 如果是 SLO 發起者時，最後要轉址過去的 URL。

key 就是 SP 對 IDP 發起 SLO 請求時，網址所帶的 return_to 參數所設定的值，而 value 就是要轉過去的 URL。

#### `.env`

設定當 SP 發起 SLO 請求之後，將 IDP 的使用者也登出。

```
LOGOUT_AFTER_SLO=true
```

### 修改 metadata

[Saml2 Service Provider](https://github.com/SocialiteProviders/Saml2) 進行 SLO 時，需要 IDP 的 metadata 提供 SingleLogoutService 的 Location，這在 [Laravel SAML IdP](https://github.com/codegreencreative/laravel-samlidp) 的預設 metadata XML 樣版中是不存在的，因此我們需要自行添加這部份。

首先需將 [Saml2 Service Provider](https://github.com/SocialiteProviders/Saml2) metadata view 加到 resources/views 下，加入 SingleLogoutService 那行：

```bash
php artisan vendor:publish --tag="samlidp_views"
```

然後修改 `resources/views/samlidp/metadata.blade.php`

```xml
    ...

    <md:SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" Location="{{ url(config('samlidp.login_uri')) }}" />
    <md:SingleLogoutService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" Location="{{ url('saml/logout') }}" ResponseLocation="{{ url('saml/logout') }}" />
  </md:IDPSSODescriptor>
</md:EntityDescriptor>
```

修改過 metadata 之後，記得清除 SP 端的 cache，因為 SP 請求過 metadata 後會 cache 住，避免重覆訪問，因此如果沒有清除 cache，SP 不會取得修改過的 metadata 資料。

### 建立 Login 相關的 route, controller 及 view

#### `routes/web.php`

```php
Route::middleware('guest')->group(function () {
  Route::get('login', [SamlController::class, 'login'])->name('login');
  Route::post('login', [SamlController::class, 'logining']);
});
```

#### `app/Http/Controllers/SamlController.php`

```php
  function login()
  {
      return view('saml/login');
  }

  function logining(Request $request)
  {
    $request->validate([
        'email' => 'required',
        'password' => 'required',
    ]);

    Auth::attempt($request->only(['email', 'password']));
    if (!Auth::check()) {
        return redirect()->route('login', $_GET)->withErrors(['certification' => "Invalid"]);
    }
  }
```

login route 必須與 `config/samlidp.php` 中的 `login_uri` 設定一致。[Laravel SAML IdP](https://github.com/codegreencreative/laravel-samlidp) 會自動監視這個 route，在 SP 發起請求時，根據登入狀態做出相應的回應。

如果使用者未曾登入，頁面會自動轉到 login route 所設定的 controller method，在這邊我們需要顯示 login form，並在 login form 中加入 saml 的 directive

#### `resources/views/saml/login.blade.php`

```
<form ... >
  @csrf
  @samlidp
```

當使用者登入成功之後，頁面將會自動轉回 SP 的 ACS URL。

## Laravel 11 相關問題

這是我第一個使用 Laravel 11 的專案，因為並沒有先詳細閱讀文件，所以一開始遇到一些陌生的情況。

Laravel 11 預設是用 database 來存放 session 及 cache，並且在 .env 中預設使用 sqlite 做為 database driver。
而 Laravel 11 也將 cache 及 session table 的 create 放進 users table 的 migration 檔案中。並在 composer create-project 之後，就跑過 migrate，產生了這些資料表。

因此一開始遭遇到 database connect 的問題時讓我大感困惑，因為在 Laravel 10 之前，創立專案之後，並不會馬上就有連接資料庫的動作，後來才發現是 .env 預設 database driver 設定的關係。

另外在我在 IDP 上實作簡易的登入功能之後，發現登入狀態一直無法保持，只要登入成功後一轉址，登入成功的 session 就消失變成未登入狀態，session key 也變了。

如果訪問其他頁面，session key 會一直保持，但是只要有 Auth::attempt 或是 Auth::login 的動作，下一次轉換頁面，session 就會重置。

後來發現只要用 database (mariadb) 做為 session driver 就會發生這種情形，改成 file 或 redis 就能正常保持登入狀態，因此我把 session driver 及 cache store 都改成了 redis。

這個問題到目前為止我還沒找到答案。
