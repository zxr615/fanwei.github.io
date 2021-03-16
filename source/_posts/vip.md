---
title: 改造 Vip
categories: PHP
tags:
  - Laravel
date: 2021-03-16 18:53:46
---


## 问题所在

> - 全部支付走统一的二维码生成接口，导致通过 type 区分需要的字段，支付方式越来越多，判断也越来越多，难以维护
> - 代码解构混乱，一个 `$data` 变量贯通整个方法，导致最后不知道 `$data` 变量里面什么数据，开发、排错越来越复杂
> - 异常处理，业务代码处处抛出 `Exception` 和捕获 `\Exception` ，导致真正程序的异常无法定位异常



## 伪代码

获取二维码接口

捕获了业务逻辑抛出 `\Exception`， 并返回错误给前端，如果有程序上的异常则难以排查错误

```php
public function qrcode(Request $request)
{
    try {
        $key = $this->checkVerifyType(0, 1);
        return $key;
    } catch (\Exception $e) {
        return '下单失败';
    }
}
```

单看方法名只是检查支付 `type` 是否正确, 但却把所有的事都干完了

各种数据全部塞进 `$data` 中, 其他地方调用后根本不知道 `$data`里面到底有什么数据

业务逻辑抛出 `\Exception` ，会导致正正程序上有异常时

```php
private function checkVerifyType($payType1 = 0, $payType2 = 0)
{
    $data = request()->all();

    if (!ctype_digit(strval($data['type']))) {
        throw new \Exception('type err');
    }

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



## 改造- 创建订单

> 1. 涉及支付的模块有：开通会员、充值、购买单个商品等
> 2. 开发支付流程：
>    1. 生成二维码（生成临时订单 `redis`，返回 `redis` 零时订单 `key`）
>    2. 手机端确认购买信息（展示购买商品信息）
>    3. 手机端确认支付 （通过临时订单的 `key` ，创建一条订单数据到数据库）
>
> 3. 创建订单和预览订单使用 `策略模式` 



### 按模块区分不同的下单链接

1. 开通会员： `/buy/vip`

2. 充值：`/buy/recharge`
3. 购买商品：`/buy/goods`



### 策略模式

Ref: [策略模式原来这么简单！](https://juejin.cn/post/6844903748788027400)

### 创建订单策略

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

3. 创建具体的支付策略

   ```php
   // 开通 vip
   class VipStrategy extends PayOrderStrategy
   {
       function createTemporaryOrder(Request $request)
       {
           $packageCode = $request['code'];
           $package     = app(PayOrderService::class)->getVipByCode($packageCode);
   
           // 临时订单数据
           $tmpOrder = [
               'package_cope' => $package->toArray(),
               'ip'           => $request->ip(),
               // ....
           ];
   
           return app(PayOrderService::class)->saveTemporaryOrder($tmpOrder);
       }
   }
   ```

   

    创建一个订单服务类，写一些创建订单的公共方法

   app/Http/Services/PayOrderService.php

   ```
   class PayOrderService
   {
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
           Cache::set('pay:' . $key, $tmpOrder, 3);
   
           return $key;
       }
   
   }
   ```

4. 目前的目录解构

   app/Http/Services/

   ```php
   ├── PayOrder
   │   ├── PayOrderContext.php
   │   ├── PayOrderStrategy.php
   │   └── Strategy
   │       └── VipStrategy.php
   └── PayOrderService.php
   ```

5. 现在开通 `vip` 就可以使用上面创建好的策略类了

   

   数据验证使用 `laravel` [表单请求验证](https://learnku.com/docs/laravel/5.6/validation/1372#106bee)


