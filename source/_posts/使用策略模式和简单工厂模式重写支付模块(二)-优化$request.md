# 使用策略模式和简单工厂模式重写支付模块(二)-优化$request

## 前情提要

上一篇文章 [使用策略模式和简单工厂模式重写支付模块](https://learnku.com/articles/55410) 最后提到有 [两点不足之处](https://learnku.com/articles/55410#ad6115) 后来思考了下觉得可以用一个实体类，把需要的参数都 `set` 进实体，用 `get` 方法获取参数，这样 `$request` 既可以不向下传递，使用 `get` 方法获取参数又能很明确。

## 建立实体类

1. 根据 [vipStrategy策略](https://learnku.com/articles/55410#07c383) 第三步中，提取临时订单中公共的数据添加对应的成员方法，并添加 `get` 和 `set` 方法，完成**公共实体类**

    ```php
    namespace App\Http\Services\PayOrder\Strategy;
    
    abstract class Entity
    {
        protected $ip;
        protected $packageCope;
        protected $code;
        protected $uid;
        protected $type;
    
        public function __construct(Request $request)
        {
            $this->ip          = $request->getClientIp();
            $this->code        = $request->get('code');
            $this->uid         = Auth::id();
            $this->packageCope = app(PayOrderService::class)->getVipByCode($this->code);
    
            // 为成员方法 `set` 值
            foreach ($request->all() as $fields => $value) {
                $property = 'set' . Str::studly($fields);
                if (property_exists($this, $fields)) {
                    $this->$property = $value;
                }
            }
        }
    
        public function getIp() { return $this->ip;}
        public function setIp($ip) { $this->ip = $ip;}
    
        public function getCode() { return $this->code;}
        public function setCode($code) { $this->code = $code;}
    
        public function getUid() { return $this->uid;}
        public function setUid($uid) { $this->uid = $uid;}
    
        public function getType() { return $this->type;}
        public function setType($type) { $this->type = $type;}
    
        public function getPackageCope() { return $this->packageCope;}
        public function setPackageCope($packageCope) {$this->packageCope = $packageCope;}
    }
    ```

2. 开通vip实体类

    ```php
    namespace App\Http\Services\PayOrder\Entity;
    
    class VipEntity extends Entity
    {
        // 开通vip月数
        protected $buyMonth;
    
        public function getBuyMonth() { return $this->buyMonth;}
        public function setBuyMonth($buyMonth){ $this->buyMonth = $buyMonth;}
    }
    ```

    

## 修改代码

vip 接口

```php
public function vip(Request $request)
{
    $strategy = new VipStrategy();
  
  	// 原代码将$request向下传传递
    $tmpOrderKey = (new PayOrderContext($strategy))->createOrder($request);
  	// 修改后：通过VipEntity() 构造实体
  	$vipEntity   = new VipEntity($request);
    $tmpOrderKey = (new PayOrderContext($strategy))->createOrder($vipEntity);

    return $this->data(['key' => $tmpOrderKey]);
}
```

VipStrategy.php

原来：接收vip接口传入的 `$request`  

```php
function createTemporaryOrder(Request $request)
{
    $packageCode = $request['code'];
  	$buyMonth    = $request['buy_month'];
    $package     = app(PayOrderService::class)->getVipByCode($packageCode);

    // 临时订单数据
    $tmpOrder = [
        'package_cope' => $package->toArray(),
        'type'         => PayOrderService::TYPE_VIP,
        'uid'          => 1,
        'ip'           => $request->ip(),
      	'buy_month'    => $buyMonth
        // ....
    ];
}
```

修改后：接收实体类，通过 `$entity` 可明确知道有什么参数

```php
function createTemporaryOrder(Entity $entity)
{
    // 临时订单数据
    $tmpOrder = [
        'package_cope' => $entity->getPackageCope(),
        'type'         => PayOrderService::TYPE_VIP,
        'uid'          => $entity->getUid(),
        'ip'           => $entity->getIp(),
        'buy_month'    => $entity->getBuyMonth()
        // ....
    ];

    $tmpOrderKey = app(PayOrderService::class)->saveTemporaryOrder($tmpOrder);

    return $tmpOrderKey;
}
```



## 总结

- 建立 `VipEntity` 实体类之后就可以明确参数，并且将 `$request` 留在控制器中不再向下传递。
- 第二个问题不明确 `redis` 临时订单中有什么数据也可以建立一个实体类，然后想参数 `set` 完成后就可以愉快的 `get` 了。 