# 用 Laravel 建立一個 SAML2 的 SSO 網站群

## 使用套件

### SP [Saml2 Service Provider](https://github.com/SocialiteProviders/Saml2)

### IDP [Laravel SAML IdP](https://github.com/codegreencreative/laravel-samlidp)

## 架構

已先建立三個 Laravel 11 的網站，分別是：

* IDP `auth.jk-web.com`
* SP `shop.jk-web.com`
* SP `member.jk-web.com`

## 步驟


### SP

在 sp 上安裝 [Saml2 Service Provider](https://github.com/SocialiteProviders/Saml2)

```bash
composer require socialiteproviders/saml2
```

修改 Laravel 相關檔案內容

#### `app/Providers/AppServiceProvider.php`

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

* `bootstrap/providers.php`

  ```php
  return [
    App\Providers\AppServiceProvider::class,
    SocialiteProviders\Manager\ServiceProvider::class,  // 加入此行
  ];
  ```

* 建立 cert 及 key 檔

  ```bash
  openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout sp_key.pem -out sp_cert.pem
  ```

  我是將 certifications 放在 storage/saml2 目錄下，下面 `config/services.php` 中的設定也對應此處。

  產生的憑 cert 檔需要在 IDP 的 config/samlidp.php 檔案中，針對對應的 SP 進行指定。 可以將憑證檔案直接存放在 IDP 伺服器上，或將憑證的內容以字串形式嵌入到設定檔中。
* `config/services.php`

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

* `routes/web.php`

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

* `.env`．以 shop.jk-web.com 為例
  ```bash
  SAML2_META_URL=https://auth.jk-web.com/saml/metadata
  SAML2_LOGOUT_URL=https://auth.jk-web.com/saml/logout?return_to=shop.jk-web.com
  # 在 IDP 的 SLO route 需加上 return_to 參數，讓瀏覽器頁面在輪流訪問完所有已建立 SSO session 的 SP 之後，回到發起 SLO 的 SP，此處的參數值需要與 IDP samlidp config 檔中的 sp_slo_redirects 設定對應
  ```

* `bootstrap/app.php` 因為 ACS URL 是 POST 方法，需要取消 CSRF 驗證，才能讓 IDP 成功訪問。
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