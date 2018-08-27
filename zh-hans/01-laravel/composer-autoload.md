composer自动加载原理

# 一.php自动加载原理
大家知道在php中如果想引入外部class,要使用include(),require()等。项目小的时候还好，项目大的时候大量的include不仅降低效率，而且也让代码变得异常臃肿。所以在php5中引入了`__autoload()`:当加载PHP类时，如果类所在文件没有被包含进来，或者类名出错，自动调用__autoload 函数。用户可以自己实现该函数来实现自动加载。
小例子：
```php
<?php
    funciton __autoload($class)
    {
        echo 'autoload class name:'.$class;
    }
    
    new test();
    
    //执行后输出
    //autoload class name:test
    //PHP Fatal error:  Uncaught Error: Class 'test' not found
```
`__autoload()`简单实现
```php
<?php
    function __autoload($className)
    {
        require_once $className.'Class.php';
    }

    new test1();
```
test1Class.php:
```php
<?php
    class test1{
        public function __construct()
        {
            echo 'welcome to class  test1';
       }
    }
```
从上面的例子可以看出来`__autoload()`函数做的事情就是找到要实例化的类的文件并加载他们，而文件是通过类名找的。因此只要确定了类名和对应文件的映射关系，就可以很轻松的实现自动加载，从而摆脱手动include的麻烦。

可是问题来了，`__autoload()`是全局函数且只能实现一次，当项目引入不同的包的时候，还需要实现每个包的映射规则，长此以往函数越来越复杂，而且每引入一个包都要实现一次，可以想象有多麻烦。这时候如果有个东西可以实现每个包都自己实现自己的autoload,然后统一注册管理就好了。

在php5.1中引入了spl库实现了该功能
```
spl_autoload_register：注册给定的函数作为 __autoload 的实现
spl_autoload_unregister：注销已注册的函数
spl_autoload_functions：返回所有已注册的函数
spl_autoload_call：尝试所有已注册的函数来加载类
spl_autoload ：__autoload()的默认实现
spl_autoload_extionsions： 注册并返回spl_autoload函数使用的默认文件扩展名。
```
详情请看[PHP官网-spl](http://php.net/manual/zh/function.spl-autoload-register.php)

以上就是php自动加载的简单介绍。

# 二.命名空间和psr规则
从上面可以看出来自动加载已经有了一个完美的解决方案，但是还缺什么呢？还缺少类名和文件的映射关系，让php在自动加载的时候能够找到文件。但是每个人可能都会对映射关系有自己的想法，想一想使用的时候，尤其是要看源代码的时候这样的痛苦吧。好在，php有对应的规范，这就是PSR-0和PSR-4规范。

PSR标准:
```
PSR-0 (Autoloading Standard) 自动加载标准
PSR-1 (Basic Coding Standard)基础编码标准
PSR-2 (Coding Style Guide) 编码风格向导
PSR-3 (Logger Interface) 日志接口
PSR-4 (Improved Autoloading) 自动加载的增强版，可以替换掉PSR-0了
PSR-7
```

PSR-0:

```
PSR-0
1.一个标准的 命名空间(namespace) 与 类(class) 名称的定义必须符合以下结构： \<Vendor Name>\(<Namespace>\)*<Class Name>；
2.其中Vendor Name为每个命名空间都必须要有的一个顶级命名空间名；
3.需要的话，每个命名空间下可以拥有多个子命名空间；
4.当根据完整的命名空间名从文件系统中载入类文件时，每个命名空间之间的分隔符都会被转换成文件夹路径分隔符；
5.类名称中的每个 _ 字符也会被转换成文件夹路径分隔符，而命名空间中的 _ 字符则是无特殊含义的。
6.当从文件系统中载入标准的命名空间或类时，都将添加 .php 为目标文件后缀；
7.组织名称(Vendor Name)、命名空间(Namespace) 以及 类的名称(Class Name) 可由任意大小写字母组成。
```
PSR-4标准
```
PSR-4
1、术语「类」是一个泛称；它包含类，接口，traits 以及其他类似的结构；

2、完全限定类名应该类似如下范例：
\<NamespaceName>(\<SubNamespaceNames>)*\<ClassName>
i、完全限定类名必须有一个顶级命名空间（Vendor Name）；
ii、完全限定类名可以有多个子命名空间；
iii、完全限定类名应该有一个终止类名；
iv、下划线在完全限定类名中是没有特殊含义的；
v、字母在完全限定类名中可以是任何大小写的组合；
vi、所有类名必须以大小写敏感的方式引用；

3、当从完全限定类名载入文件时：
i、在完全限定类名中，连续的一个或几个子命名空间构成的命名空间前缀（不包括顶级命名空间的分隔符），至少对应着至少一个基础目录。
ii、在「命名空间前缀」后的连续子命名空间名称对应一个「基础目录」下的子目录，其中的命名 空间分隔符表示目录分隔符。子目录名称必须和子命名空间名大小写匹配；
iii、终止类名对应一个以 .php 结尾的文件。文件名必须和终止类名大小写匹配；
4、自动载入器的实现不可抛出任何异常，不可引发任何等级的错误；也不应返回值；
```

从上面可以看出来PSR-0和PSR-4讲的都是命名空间和文件的映射规则，为什么会这样呢？下面来看一下命名空间。


在PHP中，命名空间用来解决在编写类库或应用程序时创建可重用的代码如类或函数时碰到的两类问题：
  1 用户编写的代码与PHP内部的类/函数/常量或第三方类/函数/常量之间的名字冲突。
  2 为很长的标识符名称(通常是为了缓解第一类问题而定义的)创建一个别名（或简短）的名称，提高源代码的可读性。
  PHP 命名空间提供了一种将相关的类、函数和常量组合到一起的途径。

首先命名空间名称定义    
```
非限定名称Unqualified name
名称中不包含命名空间分隔符的标识符，例如 Foo

限定名称Qualified name
名称中含有命名空间分隔符的标识符，例如 Foo\Bar

完全限定名称Fully qualified name
名称中包含命名空间分隔符，并以命名空间分隔符开始的标识符，例如 \Foo\Bar。 namespace\Foo 也是一个完全限定名称。
```    
    
    
 名称解析遵循下列规则：
```
1.对完全限定名称的函数，类和常量的调用在编译时解析。例如 new \A\B 解析为类 A\B。
2.所有的非限定名称和限定名称（非完全限定名称）根据当前的导入规则在编译时进行转换。例如，如果命名空间 A\B\C 被导入为 C，那么对 C\D\e() 的调用就会被转换为 A\B\C\D\e()。
3.在命名空间内部，所有的没有根据导入规则转换的限定名称均会在其前面加上当前的命名空间名称。例如，在命名空间 A\B 内部调用 C\D\e()，则 C\D\e() 会被转换为 A\B\C\D\e() 。
4.非限定类名根据当前的导入规则在编译时转换（用全名代替短的导入名称）。例如，如果命名空间 A\B\C 导入为C，则 new C() 被转换为 new A\B\C() 。
5.在命名空间内部（例如A\B），对非限定名称的函数调用是在运行时解析的。例如对函数 foo() 的调用是这样解析的：
    1.在当前命名空间中查找名为 A\B\foo() 的函数
    2.尝试查找并调用 全局(global) 空间中的函数 foo()。
6.在命名空间（例如A\B）内部对非限定名称或限定名称类（非完全限定名称）的调用是在运行时解析的。下面是调用 new C() 及 new D\E() 的解析过程： new C()的解析:
    1.在当前命名空间中查找A\B\C类。
    2.尝试自动装载类A\B\C。
new D\E()的解析:
    1.在类名称前面加上当前命名空间名称变成：A\B\D\E，然后查找该类。
    2.尝试自动装载类 A\B\D\E。
    
    
为了引用全局命名空间中的全局类，必须使用完全限定名称 new \C()。
```

在这里对命名空间介绍这么多，详情见[PHP官网-命名空间](http://php.net/manual/zh/language.namespaces.rules.php)。


写了半天发现怎么写也不如人家写的清晰，干脆把链接贴上~~~：

[PHP自动加载功能原理解析](http://leoyang90.cn/2017/03/11/PHP-Composer-autoload/)

[Composer的Autoload源码实现——启动与初始化](http://leoyang90.cn/2017/03/13/Composer%20Autoload%20Source%20Reading%20%E2%80%94%E2%80%94%20Start%20and%20Initialize/)

[Composer的Autoload源码实现——注册与运行](http://www.leoyang90.cn/2017/03/18/Composer-Autoload-Source-Reading-%E2%80%94%E2%80%94-Register-and-Run/)
