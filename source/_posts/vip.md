---
title: 使用策略模式和简单工厂模式重写支付模块
categories: PHP
tags:
  - Laravel
  - Alipay
date: 2021-03-18 14:44:45
---

最近接到一个涉及支付的需求，旧代码看的有点头大，所以捋了捋逻辑，看了下时间，还是足够的，所以就重写了一遍支付模块，抽空记录一下过程。

## 问题所在

- 全部支付走统一的二维码生成接口，导致需要通过 type 区分接收不同的字段，随着支付方式越来越多，参数判断越来越多，难以维护
- 代码解构混乱，一个 `$data` 变量贯通整个方法，导致最后不知道 `$data` 变量里面什么数据，开发、排错越来越复杂
- 异常处理，业务代码处处抛出 `\Exception` 和捕获 `\Exception` ，导致如果程序遇到了系统异常也不能及时的通知错误

## 改造前的一段伪代码

1. 所有业务逻辑错误也抛出 `\Exception` 异常，捕获 `\Exception` 后返回 `下单失败` 导致如果程序遇到真正错误时，无法及时排查错误
2. 单看 `checkVerifyType()` 方法名会认为只是检查支付 `type` 是否正确， 但却不是，这个方法把所有该干不该干的事都干完了
3. 传参用 `0` ，`1` 也不能明确知道是代表什么东西
4. `qrcode` 接口参数也很复杂，例：`type` = 1时，必须要 `code` 参数； `type` = 2 时，必须要 `price` 参数；`type` = 3 时  ....
5. `$data` 里面各种数据，有：请求数据，订单临时数据，订单预览数据，根据购买商品的不同又放入不同的数据，结果 `$data` 就是个大杂烩，修改起来实在一言难尽

```php
// 所有购买入口获取二维码的入口
public function qrcode(Request $request)
{
    try {
      	// ...
        $key = $this->checkVerifyType(0, 1);
      	// ...
        return $key;
    } catch (\Exception $e) {
        return '下单失败';
    }
}
```

*/Service/PayService.php

```php
public function checkVerifyType($payType1 = 0, $payType2 = 0)
{
    $data = request()->all();

    if (!ctype_digit(strval($data['type']))) {
        throw new \Exception('type err');
    }
  
  	// ..... 还有一堆的参数验证

    switch ($data['type']) {
        case 'vip':
            // ... 验证
            $data['vip_info'] = Vip::where('code', $data['code'])->first();
            break;
        case 'recharge':
            // ... 验证
            $data['money'] = $data['money'];
            break;
        // case...
    }

    // 优惠券判断
    if ($data['coupon_id']) {
        $money = Coupon::where('id', $data['coupon_id'])->value("money");
        $data['reduce'] = $money;
        // ....
    }

    // 订单预览信息
    $data['show_title'] = "购买一个会员";
    $data['show_money'] = 100;
  
    $key = "abcdefg";
    Redis::set($key, $data);

    return $key;
}
```

## 着手改造

> 1. 涉及支付的模块有：开通会员、充值、购买单个商品等
> 2. 开发支付流程：
>     1. 生成二维码（生成临时订单 `redis`，返回 `redis` 零时订单 `key`）
>     2. 手机端确认购买信息（展示购买商品信息）
>     3. 手机端确认支付 （通过临时订单的 `key` ，创建一条订单数据到数据库）
>     4. 根据临时订单的 `key` 创建订单
>     5. 拉起支付
>     6. 回调
> 3. 涉及到的设计模式
>     1. 策略模式
>     2. 简单工厂模式

### 前期准备

原来的返回格式：

```php
public function json($code, $msg, $data)
{
    return ['status' => $code, 'message' => $msg, 'data' => $data];
}
// 调用
json(200, "Ok", []);
```

虽然没什么大问题，但调用起来不太方便，也不直观，每次还需要传入一些不必要的参数，这里增加一些常用的返回方法

在 `BaseContrller` 中增加几个返回数据的方法，方便调用

```php
const SUCCESS_CODE = 200;
const SUCCESS_FAIL = 100;

protected function success($msg = 'ok', $data = [], $code = self::SUCCESS_CODE)
{
    return ['status' => $code, 'message' => $msg, 'data' => $data];
}

protected function data($data = [], $msg = 'ok', $code = self::SUCCESS_CODE)
{
    return ['status' => $code, 'message' => $msg, 'data' => $data];
}

protected function fail($msg = 'ok', $data = [], $code = self::SUCCESS_FAIL)
{
    return ['status' => $code, 'message' => $msg, 'data' => $data];
}
```

### 按模块区分不同的下单链接

> 由原来的统一 `qrcode` 链接分出 3 个接口，每个接口只需要接收自己需要的参数就行，不需要原来的 `type` 来区分参数 

1. 开通会员： `/buy/vip`
2. 充值：`/buy/recharge`
3. 购买商品：`/buy/goods`

### 创建临时订单策略

1. 创建一个订单的 `抽象策略`，定义算法的接口，所有策略必须实现临时订单的接口，

    app/Http/Services/PayOrder/PayOrderStrategy.php

    ```php
    abstract class PayOrderStrategy
    {
        abstract function createTemporaryOrder($request);
    }
    ```

2. 创建一个 `Context` 类

    app/Http/Services/PayOrder/PayOrderStrategy.php

    ```php
    class PayOrderContext
    {
        private $strategy;
    
        public function __construct(PayOrderStrategy $payOrderStrategy)
        {
            return $this->strategy = $payOrderStrategy;
        }
    
        public function createOrder(Request $request)
        {
            return $this->strategy->createTemporaryOrder($request);
        }
    }
    ```

3. 基础的策略框架已经搭建好，现在就需要具体的策略了

    > 开通 vip 策略

    `$request` 是[开通 vip](#实现开通vip接口) 接口中传入的 `$request`

    app/Http/Services/PayOrder/Strategy/VipStrategy.php

    ```php
    // 开通 vip
    class VipStrategy extends PayOrderStrategy
    {
      	// 组装临时订单的数据，然后存入 redis
      	// 这里是 vip 策略，所以只专注 vip 需要的数据就好
        function createTemporaryOrder(Request $request)
        {
            $packageCode = $request['code'];
            $package     = app(PayOrderService::class)->getVipByCode($packageCode);
    
            // 临时订单数据
           	$tmpOrder = [
                'package_cope' => $package->toArray(),
                'type'         => PayOrderService::TYPE_VIP,
                'uid'          => 1,
                'ip'           => $request->ip(),
                // ....
            ];
          
            return app(PayOrderService::class)->saveTemporaryOrder($tmpOrder);
        }
    }
    ```
4. 创建一个订单服务类，写一些创建订单的公共方法
    
    app/Http/Services/PayOrderService.php
    
    ```
    use Ramsey\Uuid\Uuid;
    
    class PayOrderService
    {
        const TYPE_VIP      = 1; // 购买 vip
        const TYPE_RECHARGE = 2; // 充值
        const TYPE_GOODS    = 3; // 购买商品
    
        // 通过 code 查询 vip 套餐信息
        public function getVipByCode(string $code)
        {
            // 这里应是从数据库获取数据返回
            return collect(['id' => 1, 'code' => 'vip1', 'price' => 100, 'vip_day' => 30]);
        }
    
        // 保存临时订单
        public function saveTemporaryOrder(array $tmpOrder)
        {
            $key = Uuid::uuid4()->toString();
            Cache::set($key, $tmpOrder, 3);
    
            return $key;
        }
    }
    ```

    目前的目录解构

    app/Http/Services/
    
    ```php
    ├── PayOrder
    │   ├── PayOrderContext.php
    │   ├── PayOrderStrategy.php
    │   └── Strategy
    │       └── VipStrategy.php
    └── PayOrderService.php
    ```

### 实现开通vip接口

所有接口的数据都是通过 `laravel` [表单请求验证](https://learnku.com/docs/laravel/5.6/validation/1372#106bee)

路由：routes/web.php

```php
Route::get('/buy/vip', "PayController@vip")->name('vip');
```

app/Http/Controllers/PayController.php		

```php
public function vip(Request $request)
{
    $strategy = new VipStrategy();
    $tmpOrderKey = (new PayOrderContext($strategy))->createOrder($request);

    return $this->data(['key' => $tmpOrderKey]);
}
```

```curl
curl http://127.0.0.1:8000/buy/vip?code=vip1 | json
{
  "status": 200,
  "message": "ok",
  "data": {
    "key": "35349845-0e76-4973-b240-67e7b3cdda42"
  }
}
```

临时订单已生成，现在需要需要开发手机扫码后的预览接口

### 预览订单

正常来说预览订单是每个支付都需要有的功能，所以增加一个抽象方法

1. 在 `app/Http/Services/PayOrder/PayOrderStrategy.php` 新增一个 `preview` 的抽象方法 

    ```php
    abstract class PayOrderStrategy
    {
        // 创建临时订单
        abstract function createTemporaryOrder(Request $request);
      
      	 // 预览订单
        protected function preview(array $tmpOrder)
        {
            throw new UnsupportedOperationException("不支持的方法");
        }
    }
    ```

    你可能会好奇，这里预览订单为什么要抛出一个异常呢？因为有些第三方支付没有手机支付，只能 pc 端跳转，所以就不会涉及预览这一说

    如果定义成 `abstract` 下面继承的方法有必须实现，这个非必须的就直接定义成 `protected` 并抛出一个异常，开发的时候如果错误的调用了这个方法就会知道，当前支付方式不支持订单的预览

2. 开通 vip 策略实现 `preview` 方法，参数是临时订单的信息

    app/Http/Services/PayOrder/Strategy/VipStrategy.php

    ```php
    function createTemporaryOrder(Request $request){ /*...*/ }
    
    function preview(array $tmpOrder)
    {
        $preview = [
            'title'   => '开通会员',
            'price'   => $tmpOrder['price'],
            'vip_day' => $tmpOrder['vip_day']
        ];
    
        return $preview;
    }
    ```

3. 预览订单接口

    这个接口返回一个页面，手机扫码收可以预览并且有下单按钮

    路由：routes/web.php

    ```php
    Route::get('/buy/preview', "PayController@preview")->name('preview');
    ```

    app/Http/Controllers/PayController.php

    ```php
    // 预览订单接口
    public function preview(Request $request)
    {
        // 请求下单接口后返回的临时订单 key
        $tmpOrderKey = $request->get('key');
    
        // 获取临时订单
        $tmpOrder = app(PayOrderService::class)->getTemporaryOrder($tmpOrderKey);
    
        if (!$tmpOrder) {
            throw new TemporaryOrderException("订单已过期");
        }
    
        $strategy = new VipStrategy();
        $preview = (new PayOrderContext($strategy))->preview($tmpOrder);
    
        return view('preview', $preview);
    }
    ```

4. 生成二维码

    前端请求 vip 接口之后使用返回的临时订单 key，作为 `query` 参数请求 `预览订单` 接口

    `http://127.0.0.1:8000/buy/preview?key=35349845-0e76-4973-b240-67e7b3cdda42`

     <div center="left">
         <img src="https://cdn.jsdelivr.net/gh/zxr615/md-images/images/2020image-20210317142724525.png" alt="下单二维码" width=150 />
         <img src="https://cdn.jsdelivr.net/gh/zxr615/md-images/images/2020image-20210317142241260.png" alt="手机确认支付页面" width=150 />
     </div>


5. 接下来就是立即支付了，但立即支付前还有个问题，预览订单接口的 `策略选择` 似乎有点问题，这里设计的预览订单接口不论是 `开通 vip` 还是 `充值` 都是请求这个接口，所以这里还需要判断一下购买的类型来调用不同的策略。

    这里用临时订单中的 `type` 来判断支付的类型，根据 `type` 来选择策略

    ```php
    public function preview(Request $request)
    {
        ...
        
        $strategy = new \stdClass();
        switch ($tmpOrder['type']) {
            case PayOrderService::TYPE_VIP:
                $strategy = new VipStrategy();
                break;
            case PayOrderService::TYPE_RECHARGE:
                // $strategy = new RechargeStrategy();
                break;
            // ...
        }
    
        $preview  = (new PayOrderContext($strategy))->preview($tmpOrder);
    
        return view('preview', $preview);
    }
    ```

    好嘛~，问题又来了，这 `switch` 看着有点不爽，再把它独立出来吧，加一个获取策略的 `简单工厂` ，接下来优化这段选择策略的代码

    创建 `PreviewFactory` 简单工厂

    `touch app/Http/Services/PayOrder/PreviewFactory.php`

    ```php
    namespace App\Http\Services\PayOrder;
    
    use App\Exceptions\BusinessException;
    use App\Exceptions\TemporaryOrderException;
    use App\Http\Services\PayOrder\Strategy\VipStrategy;
    use App\Http\Services\PayOrderService;
    
    class PreviewFactory
    {
        public static function strategy(string $key)
        {
            // 获取临时订单
            $tmpOrder = app(PayOrderService::class)->getTemporaryOrder($key);
    
            if (!$tmpOrder) {
                throw new TemporaryOrderException("订单已过期");
            }
    
            $strategy = new \stdClass();
            switch ($tmpOrder['type']) {
                case PayOrderService::TYPE_VIP:
                    $strategy = new VipStrategy();
                    break;
                case PayOrderService::TYPE_RECHARGE:
                    // return new Recharge();
                    break;
                // ...
                default:
                    throw new BusinessException('订单类型错误.');
            }
    
            return $strategy;
        }
    }
    ```

    现在再来看看 `preview` 接口

    ```php
    public function preview(Request $request)
    {
        $tmpOrderKey = $request->get('key');
    
        try {
            $preview = PreviewFactory::strategy($tmpOrderKey)->preview($tmpOrderKey);
        } catch (TemporaryOrderException $e) {
            return $this->fail($e->getMessage());
        }
    
        return view('preview', $preview);
    }
    ```


### 发起支付

1. 和创建订单策略同样，我们也创建一个支付策略

    /Users/tuju/Project/pay/app/Http/Services/Payment

    ```tree
    Payment
    ├── PaymentContext.php
    ├── PaymentFactory.php
    ├── PaymentStrategy.php
    └── Strategy
        ├── AlipayStrategy.php
        ├── UnionStrategy.php
        └── WechatStrategy.php
    ```

2. 定义支付接口

    PaymentStrategy.php

    ```
    interface PaymentStrategy
    {
        public function pay(array $order);
    } 
    ```

3. 上下文联系

    PaymentContext.php

    ```php
    class PaymentContext
    {
        private $strategy;

        public function __construct(PaymentStrategy $paymentStrategy)
        {
            return $this->strategy = $paymentStrategy;
        }

        public function pay(array $order)
        {
            return $this->strategy->pay($order);
        }
    }
    ```

4. 获取支付策略的工厂

    PaymentFactory.php

    ```php
    class PaymentFactory
    {
        public static function strategy(string $payType)
        {
            switch ($payType) {
                case 'wechat':
                    $strategy = new WechatStrategy();
                    break;
                case 'alipay':
                    $strategy = new AlipayStrategy();
                    break;
                case 'union':
                    $strategy = new UnionStrategy();
                    break;
                // case...
                default:
                    throw new BusinessException("支付方式不存在");
            }

            return $strategy;
        }
    }
    ```
5. 制定具体支付策略
    1. 支付宝策略
        Strategy/AlipayStrategy.php

        ```php
        class AlipayStrategy implements PaymentStrategy
        {
            public function pay(array $order)
            {
                /**
                * 向支付宝请求
                * @see 支付宝官方sdk https://github.com/alipay/alipay-easysdk/tree/master/php
                * @see 第三方sdk https://github.com/lokielse/omnipay-alipay
                */
                return "https://www.alipay.com/";
            }
        }
        ```
    2. 微信策略
    Strategy/WechatStrategy.php

    ```php
    class WechatStrategy implements PaymentStrategy
    {
        public function pay(array $order)
        {
            /**
            *
            * @see 微信官方 https://github.com/wechatpay-apiv3/wechatpay-guzzle-middleware
            * @see 官方文档 https://pay.weixin.qq.com/wiki/doc/apiv3/open/pay/chapter2_6_2.shtml
            * @see 第三方sdk https://github.com/lokielse/omnipay-wechatpay
            */
            return "https://pay.weixin.qq.com/";
        }
    }
    ```

6. 确认支付接口

    app/Http/Controllers/PayController.php

    ```php
    public function pay(Request $request)
    {
        $tmpOrderKey = $request->get('key');
        // pay_type=wechat|alipay|union
        $payType = $request->get('pay_type');

        // 处理订单数据、创建订单
        $tmpOrder = app(PayOrderService::class)->getTemporaryOrder($tmpOrderKey);
        $order = [ /** ... */];
        $created = app(PayOrderService::class)->createOrder($order);

        if (!$created) {
            return $this->fail("支付失败, 请重新生成订单.");
        }

        // 发起支付
        try {
            // 前面我们定义了一个支付策略工厂模式，帮助我们实例化策略，所以这里传入我们的支付方式
            // 工厂就会帮我们对应支付策略返回回来，然后我们再统一调用 pay() 这个方法
            $strategy = PaymentFactory::strategy($payType);
            // 一般第三方会返回一个支付跳转链接，点击确认支付的时候用户是已经在手机页面了
            // 所以直接跳转链接就可以拉起对应的支付了。
            $url = (new PaymentContext($strategy))->pay($created);
        } catch (BusinessException $e) {
            $this->fail($e->getMessage());
        }

        // 跳转
        return redirect($url);
    }
    ```

7. 最后列出下最终策略模块的树状图

    ```
    ├── PayOrder 支付相关策略集合
    │   ├── PayOrderContext.php
    │   ├── PayOrderStrategy.php
    │   ├── PreviewFactory.php
    │   └── Strategy 具体策略，如果要充值，则新建一个充值策略即可，新增的方式也不会影响到开通会员的相关功能
    │       └── VipStrategy.php 开通 vip 策略
    ├── PayOrderService.php 一些公用方法
    └── Payment 支付相关策略集合
        ├── PaymentContext.php
        ├── PaymentFactory.php
        ├── PaymentStrategy.php
        └── Strategy 具体策略，可以增加各种第三方支付
            ├── AlipayStrategy.php 支付宝
            ├── UnionStrategy.php 微信
            └── WechatStrategy.php 银联
    ```



## 解决的问题

- 将接口细分，不再是所有订单都进入同一个方法，解决了参数混乱问题

- 把大杂烩 `$data` 中的数据全部切分到每个不同的策略中去，而不是在方法中使用大量的 `if` 和 `switch` 来处理，再增加类型时只需要关注新增的策略即可

- 把接口数据用 `laravel` [表单请求验证](https://learnku.com/docs/laravel/5.6/validation/1372#106bee) 来判断，而不是在 `controller` 和  `service` 层用 `if` 来判断

- 把 `\Exception` 代码全部替换成相应的业务异常

  ​    

## 不足之处

- `$request` 我认为还是不要往下传递比较好，最好在控制器中处理，但整体支付逻辑还是比较复杂，传参的话又需要传入很多参数，暂时也没有想出什么好的方法，所以还是决定将 `$request` 往下传递了。

- 创建订单中的零时订单存入到 redis 后再获取，还是不能明确知道数组里具体存入了什么数据，在 GO 中在序列化 `json` 时需要一个 `struct` 来支持，明确表名 json 中有什么字段，这样开发时既不容易出错，也减少很多梳理代码的时间；我认为可以新建一个 class 来模拟 GO 中的 `struct` 来明确 json 里面有什么数据。

    

## 总结

1. 不要抛出 `\Exception` 异常，业务上的错误异常应该抛出自定义异常

2. 尽量不要去捕获 `\Exception` 异常，`\Exception` 异常应该由顶层的 `Handel` 去处理；如遇到事务需要 `rollback` 的话，捕获 `Exception` 后，在返回错误信息前，需要手动记录下异常的详细信息。

3. 一段代码如果有两处以上用到，应该独立出一个公共方法。

4. 参数的验证在控制层面就校验完成，不要再传到 service 中处理。

5. 不要用 `0`, `1`, `2` 传参、判段等，不梳理上下文代码，实在是不知道什么意思，如果改变了其代号意思，则所有涉及到的地方都需要修改判断，可以用常量来管理各种代号。

    ``` php
    public const PAY_STATUS_FAIL = 0;
    public const PAY_STATUS_OK   = 1;
    public const PAY_STATUS_WAIT = 2;
    
    public function give($payStatus)
    {
        // 最不明确的方法, 如果没注释真不知道什么意思
        if ($payStatus == 1) {
            // ...
        }
    
        // 比较好的方法，即便没有注释，意思也比较明确
        if ($payStatus == self::PAY_STATUS_OK) {
            // ...
        }
    
        // 我更喜欢用的方法，定义一个方法，看方法名知其意
        if ($this->isPaid($payStatus)) {
            // ...
        }
    }
    
    // 是否已支付完成
    public function isPaid($payStatus)
    {
        return $payStatus == self::PAY_STATUS_OK;
    }
    ```

6. 善用设计模式

最后，大家有什么改进之处，或者疑问之处欢迎大家提出、指正。