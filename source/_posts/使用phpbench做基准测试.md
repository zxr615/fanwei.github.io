---
title: 使用phpbench做基准测试
categories: PHP
tags:
  - Test
  - Benchmark
date: 2021-03-19 11:31:28
---

## 项目地址
```url
https://github.com/thephpleague/omnipay
```

## 文档地址

[https://phpbench.readthedocs.io/en/latest/quick-start.html](https://phpbench.readthedocs.io/en/latest/quick-start.html)

### 安装phpbench

````console
composer global require phpbench/phpbench --dev -vvv
````

### 创建phpbench.json 配置文件

在项目的根目录中创建一个文件：`phpbench.json`

[PHPBench配置](https://phpbench.readthedocs.io/en/latest/quick-start.html#phpbench-configuration)

```json
{
    "bootstrap": "vendor/autoload.php"
}
```

### 修改composer.json

```json
"autoload-dev": {
    "psr-4": {
        "Acme\\Tests\\": "tests/"
    }
},
```

### 更新 autoloader

`composer du`

## 编写Bench代码

方法必须以 `bench` 开头

```php
class Test
{
    public function benchTest()
    {
        usleep(300);
    }
}
```

### 执行测试

```
phpbench run tests/Benchmark/Test.php --report=default
```

## 结果

```console
➜  pay phpbench run tests/Benchmark/Test.php --report=default
PhpBench @git_tag@. Running benchmarks.
Using configuration file: /Users/tuju/Project/pay/phpbench.json

\Tests\Benchmark\Test

    benchConsume............................I0 [μ Mo]/r: 481.000 481.000 (μs) [μSD μRSD]/r: 0.000μs 0.00%

1 subjects, 1 iterations, 1 revs, 0 rejects, 0 failures, 0 warnings
(best [mean mode] worst) = 481.000 [481.000 481.000] 481.000 (μs)
⅀T: 481.000μs μSD/r 0.000μs μRSD/r: 0.000%
suite: 134628febd27af811e3ceaa765bccf4c891bc2f4, date: 2021-03-19, stime: 04:11:57
+-----------+--------------+-----+------+------+------------+-----------+--------------+----------------+
| benchmark | subject      | set | revs | iter | mem_peak   | time_rev  | comp_z_value | comp_deviation |
+-----------+--------------+-----+------+------+------------+-----------+--------------+----------------+
| Test      | benchConsume | 0   | 1    | 0    | 4,270,848b | 481.000μs | 0.00σ        | 0.00%          |
+-----------+--------------+-----+------+------+------------+-----------+--------------+----------------+

```

## 注意

使用 `composer` 命令需要将 `composer` 的 `bin` 目录给 `export` 出来

```console
export PATH="$HOME/.composer/vendor/bin":$PATH
```

