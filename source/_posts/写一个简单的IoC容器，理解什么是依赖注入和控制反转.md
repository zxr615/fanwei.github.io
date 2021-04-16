---
title: 写一个简单的IoC容器案例，理解什么是依赖注入和控制反转
categories: PHP
tags:
  - Laravel
  - IoC
  - DI
date: 2021-04-07 17:35:54
---

## 简介

依赖注入(DI)，控制反转(IoC)，容器(Container) 经常都经常会提到，但很长一段时间都是一知半解，现在抽空把自己浅显理解的内容记录下来，与大家探讨。



## 引子
不知道大家开发时有没有好奇过以下这两个问题呢？
1. 为什么方法的参数位置①是需要传入 **2** 个参数的，一个是 `Request` 类型的参数，一个是不定类型的 `$id` 参数，但路由只有一个 `$id` 参数，那 `$request` 参数是哪里来的？
2. `UserService` 的 `__construct` 方法明确实例化需要一个 `Cache` 类型的参数，但②中并没有传入，为什么能使用呢？③为什么使用 `new` 不传参数就会报错呢？

```php
Route::get('/{id}','\App\Http\Controllers\IndexController@index');
```

```php
class IndexController extends Controller
{
    public function index(Request $request, $id)①
    {
		app(UserService::class)②->getUserNameById($id);
      
        ③// TypeError: Too few arguments to function App/Services/UserService::__construct(), 0 passed in Psy Shell code on line 1 and exactly 1 expected
        (new UserService())->getUserNameById($id);
    }
}
```

```php
class UserService
{
    public $cache;

    public function __construct(Cache $cache)
    {
        $this->cache = $cache;
    }

    public function getUserNameById($id)
    {
        return $this->cache->get('user:id:' . $id);
    }
}
```



原来这叫 `依赖注入` ，开始也不知道是个啥，那就抱着这两个疑问开始寻找答案。 




很早之前就很好奇：

1. 为什么方法的参数位置①是需要传入 **2** 个参数的，一个是 `Request` 类型的参数，一个是不定类型的 `id` 参数，但路由只有一个 `id` 参数，那 `$request` 参数是哪里来的？
2. `UserService` 的 `__construct` 方法明确实例化需要一个 `Cache` 类型的参数，但②中并没有传入，为什么能使用呢？③为什么使用 `new` 如果不传参数就会报错呢？

不知道大家开发时有没有好奇过这两个问题呢？后来听说这叫 `依赖注入` ，也不知道是个啥，那就抱着这两个疑问开始寻找答案。 



## 贯穿全文

接下来会围绕这 3 个点来讲

1. 依赖控制
    1. 依赖：谁依赖谁
    2. 注入：注入什么

2. 控制反转

    1. 控制：谁控制谁
    2. 反转：反转什么

3. 什么是容器

    

## 常规代码

Controller1

```php
class Index1Controller
{
    public $userService;

    public function __construct() {
        /**
         * 因为我需要(依赖) UserService() 给我提供数据, 所以创建了一个 UserService() 对象
         *
         * 控制：我 (IndexController) 控制了 UserService() 对象的创建
         * 反转：我 (IndexController) 绝对控制 UserService() 对象的权利，创建对象的控制权没有发生转移，所以没有反转，一切都是亲力亲为。
         */
        $this->userService = new UserService();
    }

    public function index() {
        // 我 (index) 控制了 UserService() 对象的创建
        $userService = new UserService();

        $userName = $userService->getUserName();
        $userName2 = $this->userService->getUserName();

        return [$userName, $userName2];
    }
}

(new IndexController())->index();
```

Index2.php

```php
<?php
	(new Index1Controller())->index();
```

生活比喻：

```
依赖：我要吃面包，面包需要(依赖)面粉才能制作

注入：买面粉 -> 注入水 -> 制作面包 -> 吃

控制：我控制了面包的制作

反转：无
```



  ## 依赖注入和控制反转

Controller2

```php
class Index2Controller
{
    public $userService;

    /**
     * 因为我需要(依赖) UserService() 给我提供数据, 所以我需要接收一个 UserService 类型的参数
     * 把依赖从外部传入进来，把需要的依赖传入进来了，就是依赖注入
     *
     * 控制：调用者控制了 UserService() 对象的创建
     * 反转：我 (IndexController) 控制 UserService 创建的权利已经没有了(转移了)，那转移给谁了？这里的控制权转移给调用者了。
     */
    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }

    public function index()
    {
       /**
         * 在方法中创建对象
         * 我 (index) 控制了 UserService() 对象的创建
         */
        $userService = new UserService();
        $userName  = $userService->getUserName();
      
        $userName2 = $this->userService->getUserName();

        return [$userName, $userName2];
    }
}

// __construct() 中创建 new UserService() 转移到了这里
$userService = new UserService();
// 将 $userService 传入(注入) controller 中
(new Index2Controller($userService))->index();
```

Index2.php

```php
<?php
// __construct() 中创建 new UserService() 转移到了这里
$userService = new UserService();
// 将 $userService 传入(注入) controller 的构造函数中
$rs = (new Index2Controller($userService))->index();
var_dump($rs);
```

生活比喻：

```php
依赖：我要吃面包，依赖面包店

注入：告诉面包店老板要吃什么 -> 老板给你(注入) -> 吃

控制：面包店老板控制面包的制作

反转：原来我控制面包的制作的权利没有了，转移给了面包店的老板
```



## IoC 容器自动注入

上面的 `依赖注入和控制反转` 并没有解决开头引出的两个问题的答案，依赖还是需要手动创建，然后手动注入，如何实现依赖的自动注入呢？这个时候就需要一个 IoC 容器了

- 如何注入

    使用 `PHP` 提供的 [反射(Reflection)](https://www.php.net/manual/en/book.reflection.php) 功能

- 我们需要注入哪里的参数

    依赖注入是以构造函数参数的形式传入，所以我们需要自动注入构造函数指定的参数

- 我们需要注入哪些参数

    我们只注入类实例，其他参数原样传入

## Container

IoC 容器其实就是一个普通的 `class` 类，实现了某些功能而已，不必想的太复杂。

```php
class Container
{
  	// 在 laravel 中这个方法是 `make()`, 这里为了方便和常用的 new xxx() 理解，所以命名成了「自动注入的new」
    public static function autoInjectNew($className, $params = [])
    {
        $reflect = new \ReflectionClass($className);
        // 获取构造函数
        $construct = $reflect->getConstructor();

        // 保存实例化需要的参数
        $args = [];
        if ($construct) {
            /**
             * 获取构造函数的参数
             * array(2) {
             *  [0] => object(ReflectionParameter)#3 (1) {["name"]=> string(11) "userService"}
             *  [1] => object(ReflectionParameter)#4 (1) {["name"]=> string(3) "uid"}
             *  }
             */
            $consParams = $construct->getParameters();
            foreach ($consParams as $param) {
                $class = $param->getClass();
                if ($class) {
                    // $args[] = new $class->name();
                    // 如果这样处理依赖的的 UserService() 还有依赖的话则无法兼顾，所以需要递归处理

                    // demo 中这里相当于就是 new Study\Di\Services\UserService()
                    $args[] = self::autoInjectNew($class->name);
                }
            }
        }

        // 合并参数
        $args = array_merge($args, $params);

        /**
         * IoC 控制反转:
         *  控制：容器控制了对象的创建
         *  反转：创建对象的权利已经转移到了容器中来了，不再是 IndexController() 中的 __construct() 了。
         * DI 依赖注入:
         *  依赖：$args 保存了保存了需要那些依赖
         *  注入：把 $args 中的依赖作为参数传入(注入)，返回实例
         */
        // 相当于：$instance = new Index3Controller(new UserService)
        $instance = $reflect->newInstanceArgs($args);

        return $instance;
    }
}
```

验证一下

Controller3

```php
class Index3Controller
{
    protected $userService;

    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }

    public function index()
    {
        $userName = $this->userService->getUserName();
        return $userName;
    }
}
```

index3.php

```php
<?php
$index3Instance = Container::autoInjectNew(Index3Controller::class);
$rs = $index3Instance->index();
var_dump($rs);
```

现在再看看是不是没有主动传入 `new UserService()` 参数也可以成功调用啦

## 回顾问题

1. 路由中的 `Request $request` 参数是哪里来的

    答：请求进入框架之后，框架解析 `url` 找到相对应的控制器类，调用容器写好的自动注入方法(案例中是`autoInjectNew()`)，进行注入参数，这样就可以愉快又方便的使用啦。

2. 使用 `app()` 和 `new` 有什么不同

    答：其实 `laravel` 中 `app()` 就是使用 `Container` 实例化的一个助手函数，我们可以来写一个助手函数

    先看看 `laravel` 中的助手函数

    ```php
    function app($abstract = null, array $parameters = [])
    {
        if (is_null($abstract)) {
            return Container::getInstance();
        }
    
      	// 这里的 make 就相当于当前项目中的 autoInjectNew()
        return Container::getInstance()->make($abstract, $parameters);
    }
    ```

    实现助手函数 app()

    index3.php

    ```php
    <?php
    $index3Instance = Container::autoInjectNew(Index3Controller::class);
    $rs = $index3Instance->index();
    var_dump("indexRs: ", $rs);
    
    // 使用助手函数
    $appRs = app(Index3Controller::class)->index();
    var_dump("appRs: ", $appRs);
    
    // 助手函数
    function app($class, $params = []) {
        return Container::autoInjectNew($class, $params);
    }
    ```



## 总结

刚开始的时候在网上找了很多相关的文章，但看下来说的似乎都大同小异，但还是不理解，很是苦恼。经常看到「服务容器是 Laravel 的核心」这样的说法，所以就去从 `laravel` 的 `index.php` 开始一步一步过，但 `laravel` 的源码看的确实也有点头大，所以我转了个弯，把 `ThinkPHP` 的的框架 `clone` 下来看了看，确实看的轻松许多，再回头看 `laravel` 的源码，还是很复杂，但理解起来相对直接看 `laravel` 就简单多了。

文章很多都是作者自己的理解，文章提供的大多也只是很少一部分的代码，要弄清楚还是得阅读源码。

这个案例的 `Container` 中似乎没有太体现出 `容器` 这个词，因为还没有实现实例化对象的存储，具体可以看看相关的源码。



## 案例demo

[https://github.com/zxr615/study-ioc](https://github.com/zxr615/study-ioc)



## 参考

[https://github.com/top-think/framework/blob/6.0/src/think/Container.php](https://github.com/top-think/framework/blob/6.0/src/think/Container.php)

[https://github.com/laravel/framework/blob/8.x/src/Illuminate/Container/Container.php](https://github.com/laravel/framework/blob/8.x/src/Illuminate/Container/Container.php)

[https://segmentfault.com/a/1190000018948909](https://segmentfault.com/a/1190000018948909)

[https://blog.csdn.net/bestone0213/article/details/47424255](https://blog.csdn.net/bestone0213/article/details/47424255)

[https://www.cnblogs.com/DebugLZQ/archive/2013/06/05/3107957.html](https://www.cnblogs.com/DebugLZQ/archive/2013/06/05/3107957.html)

