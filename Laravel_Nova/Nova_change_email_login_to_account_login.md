# Nova 修改登入欄位

目標： 信箱登入 改成 帳號登入

1. 資料表要建立相對欄位
2. 找到 /vendor/laravel/ui/auth-backend/AuthenticatesUsers.php

```bash
public function username()
    {
        return 'account';  #修改你要用來登入的欄位
    }
```

目標： 讓帳號有狀態，如果狀態關閉會拒絕登入

1. 自己建立一個 middleware

```bash
public function handle(Request $request, Closure $next)
    {
        $account = $request->input('account');
        $user = User::where('account', $account)->first();
        if ($user && !$user->is_active) {
            $request->merge(['account' => 'wrong_account']);
            return $next($request);
        }
        return $next($request);
    }
```

1. 找到 /vendor/laravel/nova/src/PendingRouteRegistration.php，把middleware加上去

```bash
Route::namespace('Laravel\Nova\Http\Controllers')
            ->domain(config('nova.domain', null))
            ->middleware($middleware)
            ->prefix(Nova::path())
            ->group(function (Router $router) {
                $router->get('/login', [LoginController::class, 'showLoginForm'])->name('nova.pages.login');
            });

        Route::namespace('Laravel\Nova\Http\Controllers')
            ->domain(config('nova.domain', null))
            ->middleware(['checkUserStatus','nova'])
            ->prefix(Nova::path())
            ->group(function (Router $router) {
                $router->post('/login', [LoginController::class, 'login'])->name('nova.login');
            });
```

參考

1. [https://stackoverflow.com/questions/51990187/how-to-login-using-username-instead-of-email-in-laravel-nova](https://stackoverflow.com/questions/51990187/how-to-login-using-username-instead-of-email-in-laravel-nova)