# MonoLog使用说明
>@author owen

#### Monolog是一个符合PSR标准的PHP日志库包。
#### 此使用说明是本人使用Monolog一段时间后的理解，整理一篇文档。

>最好的文档应该是官方文档。作为了一个技术最应多读的是官方文档

## 先给出一个DEMO，

```php

// 实例化一个具体的日志处理类 ，设定要记录的日志等级
// monolog自带的日志处理类有20个左右，在monolog\src\Monolog\Handler下
$logName = '/var/logs/test_'.date('Ymd').'.log';
$stream = new StreamHandler($logName, Logger::DEBUG);

// 实例化日志类，并且把具体的处理类注入到日志类中
$logger = new Logger($channel);
$logger->pushHandler($stream);

//添加extra扩展字段的信息
// $logger->pushProcessor(function ($record) { 
//     $record['extra']['dummy'] = 'Hello world!'; 
//     return $record;
// });

// 记录不同等级的日志
$logger->debug('msg', ['aa'=>'123', 'bb'=>567]);
$logger->info('msg', []);

```

### 日志等级说明
monolog日志分8个等级，由低到高分别为：  
DEBUG,INFO,NOTICE,WARNING,ERROR,CRITICAL,ALERT,EMERGENCY

### Handler设置日志等级说明
Handler是具体日志操作类，第一个参数为日志文件名，第二个参数为最低记录等级  
如第二参数传Logger::NOTICE，那么$logger->info, $logger->debug不会记录日志信息，大于等于Logger::NOTICE等级的日志会记录在内。

## Monolog 模块说明

Monolog分两大块内容， Logger和Handler  
Logger负责判断日志等级，设定日志格式，调度一个或者多个Handler  
所有的Handler实现HandlerInterface接口，提供接口方法。  
不同的Handler实现往不同的地方存放日志，如把日志放到文件，发送到TCP远程，发送到邮箱，发送到ElasticSearch等。  
  
Handler依赖注入到Logger中，可以自已实现一个xxxHandler实现HandlerInterface接口的方法既可注入Logger.  
  
## Monolog 特点描述

    -   PSR标准化
    -   组件化，可自已实现Handler注入(官方自带Handler满足绝大多数需求)
    -   可添加多个Handler，同一条日志记录发送到多个不同的地方
    -   可添加多个Processor，一条记录可无限扩展相关信息
    -   自动多创建多级目录，日志容易分模块
    -   composer安装，方便使用，不需要添加PHP扩展

##　灵活用法

```
$stream１ = new StreamHandler($logName1, Logger::DEBUG);
$stream２ = new StreamHandler($logName2, Logger::NOTICE);

$logger = new Logger($channel);
$logger->pushHandler($stream1);
$logger->pushHandler($stream2);

// INFO>Logger::DEBUG, INFO<Logger::NOTICE
$logger->info('msg1'); //只有stream1会记录此条记录
// ERROR>Logger::DEBUG, ERROR>Logger::NOTICE
$logger->ERROR('msg2');// stream1, stream2都会记录此条记录

// 添加多个扩展信息就不举例了，相信你已经知道怎么写了。

```

如果你还在用自已写的file_put_contents，那就试着用用Monolog吧。