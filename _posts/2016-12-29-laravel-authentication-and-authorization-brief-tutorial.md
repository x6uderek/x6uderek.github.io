---
layout: page
title: laravel认证与权限系统简明教程
date: 2016-12-29 12:49:00
categories: blog
tags: [laravel,middleware]
description: laravel认证与权限系统简明教程
---


laravel的认证和授权系统很强大，很全面，但是我们一般不会用到那么多的功能，我总结了一下我自己经常用的一些开发流程。

注意：本文基于**laravel5.3**版本编写

## 一、用户登录

无论是用户登录还是权限检查都是通过中间件实现的。中间件可以在路由或路由组上配置，也可以在`App\Http\Kernel`里面配置。`routes/api.php`下的路由已经应用了api中间件组，`routes/web.php`下面的路由已经应用了web中间件组。这里以后台用户登录为例说明登录流程。

首先建立一个后台用户模型AdminUser，这个类需要实现Authenticatable接口。
```php
class AdminUser implements Authenticatable
{
    private $_data;
    public function __construct($id = '') {
        if($id) {
            $this->_data = self::get($id);
        }
    }

    /**
     * Get the name of the unique identifier for the user.
     *
     * @return string
     */
    public function getAuthIdentifierName()
    {
        return 'id';
    }

    /**
     * Get the unique identifier for the user.
     *
     * @return mixed
     */
    public function getAuthIdentifier()
    {
        return $this->_data->id;
    }

    /**
     * Get the password for the user.
     *
     * @return string
     */
    public function getAuthPassword()
    {
        return $this->_data->password;
    }

    /**
     * Get the token value for the "remember me" session.
     *
     * @return string
     */
    public function getRememberToken()
    {
        return '';
    }

    /**
     * Set the token value for the "remember me" session.
     *
     * @param  string $value
     * @return void
     */
    public function setRememberToken($value)
    {
        return;
    }

    /**
     * Get the column name for the "remember me" token.
     *
     * @return string
     */
    public function getRememberTokenName()
    {
        return '';
    }

    public function __get($name) {
        if(!$this->_data) {
            return null;
        }
        return $this->_data->{$name};
    }

    public function __set($name, $value) {
        if(!$this->_data) {
            return;
        }
        $this->_data->{$name} = $value;
    }

    public function __isset($name) {
        return isset($this->_data->{$name});
    }

    public function __unset($name) {
        if(!$this->_data) {
            return;
        }
        unset($this->_data->{$name});
    }

    private static function &getConnection() {
        $db = DB::connection('admin')
            ->table('admin_users');
        return $db;
    }

    public static function get($id) {
        return self::getConnection()->where('id', $id)->first();
    }

    public static function getByName($name) {
        $data = self::getConnection()->where('name', $name)->first();
        if($data) {
            $result = new self();
            $result->_data = $data;
            return $result;
        }
        return null;
    }
}
```

再实现一个BackstageUserProvider，这个类要实现UserProvider接口，主要实现密码匹配逻辑。

```php
class BackstageUserProvider implements UserProvider
{

    /**
     * Retrieve a user by their unique identifier.
     *
     * @param  mixed $identifier
     * @return \Illuminate\Contracts\Auth\Authenticatable|null
     */
    public function retrieveById($identifier)
    {
        return new AdminUser($identifier);
    }

    /**
     * Retrieve a user by their unique identifier and "remember me" token.
     *
     * @param  mixed $identifier
     * @param  string $token
     * @return \Illuminate\Contracts\Auth\Authenticatable|null
     */
    public function retrieveByToken($identifier, $token)
    {
        return null;
    }

    /**
     * Update the "remember me" token for the given user in storage.
     *
     * @param  \Illuminate\Contracts\Auth\Authenticatable $user
     * @param  string $token
     * @return void
     */
    public function updateRememberToken(Authenticatable $user, $token)
    {
        return;
    }

    /**
     * Retrieve a user by the given credentials.
     *
     * @param  array $credentials
     * @return \Illuminate\Contracts\Auth\Authenticatable|null
     */
    public function retrieveByCredentials(array $credentials)
    {
        return AdminUser::getByName($credentials['name']);
    }

    /**
     * Validate a user against the given credentials.
     *
     * @param  \Illuminate\Contracts\Auth\Authenticatable $user
     * @param  array $credentials
     * @return bool
     */
    public function validateCredentials(Authenticatable $user, array $credentials)
    {
        if($user->status == 1 && $user->getAuthPassword()==md5($credentials['password'])) {
            return true;
        }
        return false;
    }
}
```
然后在`App\Providers\AuthServiceProvider`里面注册这个用户认证方式
```php
public function boot()
{
	$this->registerPolicies();

	Auth::provider(BackstageUserProvider::class, function ($app, array $config){
		return new BackstageUserProvider();
	});
}
```
在config/auth.php里面配置这个provider
```php
'guards' => [
	'backstage' => [
		'driver' => 'session',
		'provider' => 'backstage_users',
	],
],
'providers' => [
	'backstage_users' => [
		'driver' => App\Auth\BackstageUserProvider::class
	],
],
```
这时将`'auth:backstage'`这个中间件加到路由上面就能保证路由的访问用户是登录过的。
## 二、登录界面
如果用户没有登录，将会被重定向到/login这个路径，这个路径可以在`App\Exceptions\Handler`的`unauthenticated`方法里面修改。

在这个路径上面建立登录界面，用户输入用户名密码后，进行登录操作
```php
if(Auth::guard('backstage')->attempt(['name'=>$request->input('name'),'password'=>$request->input('password')])) {
	return redirect()->intended('/');
}
else {
	return view('admin/login', ['msg'=>'用户名或密码错误']);
}
```
**注意这里是`Auth::guard('backstage')`不能直接使用`Auth`**

对应的登出操作：
```php
Auth::guard('backstage')->logout();
```
## 三、权限检查

这里有两种级别的权限检查方案

1. 基于路由的权限检查，这种方式只检查用户是否具有当前路由的访问权限，与当前路由访问的资源无关。
2. 基于资源访问的权限检查，这种方式检查用户对当前操作相关的数据资源是否具有权限，更加详细。比如“用户只能编辑自己发的帖子”这种权限检查

这里可以根据自己的需要选择方案

### 1、如果选择第一种方案,首先建立一个BackstageRoutePolicy
```php
class BackstageRoutePolicy
{
    public function visitRoute($user) {
        $route_name = Route::currentRouteName();
        return $user->canVisit($route_name);
    }
}
```
然后在`App\Providers\AuthServiceProvider`配置这个policy
```php
protected $policies = [
    'App\Model\Backstage\Module' => BackstagePolicy::class,
];
```
现在只要在需要保护的路由上加上`'can:visitRoute,App\Model\Backstage\Module'`就可以了

### 2、如果选择第二种方案，首先新建一个Policy
```php
class PostPolicy
{
    public function update(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }
}
```
然后在`App\Providers\AuthServiceProvider`注册policy
```php
protected $policies = [
    Post::class => PostPolicy::class,
];
```
使用policy的4种方式

a) 使用`User::can()`
```php
$user = Auth::guard('backstage')->user();
if ($user->can('update', $post)) {
    //
}
else {
    throw new AuthorizationException('未授权',403)
}
```
b) 中间件
```php
Route::put('/post/{post}', 'PostController@edit')->middleware('can:update,post');
```
c) 使用`Controller::authorize()`
```php
class PostController extends Controller
{
    public function update(Request $request, Post $post)
    {
        $this->authorize('update', $post);
        //go on
    }
}
```
d) Blade模板
```
@can('update', $post)
@elsecan('create', $post)
@endcan
```
上面的代码等价于
```
@if (Auth::user()->can('update', $post))
@elseif (Auth::user()->can('create', $post))
@endif
```