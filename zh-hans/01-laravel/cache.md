Laravel源码解读之：Cache

Laravel中的cache支持file,apc,memcache,redis,database等多种缓存驱动。使用者可以按需选择适合自己的缓存驱动，同时也可以在多种驱动间快速切换，或者使用自定义的缓存驱动。下面看下laravel是如何实现这些的：


config/app.php中的provider里有cache的服务提供者
```php
Illuminate\Cache\CacheServiceProvider::class,
```

进一步查找代码看到CacheServiceProvider中：
```php
 public function register()
    {
        // 在容器中绑定一个缓存管理类
        $this->app->singleton('cache', function ($app) {
            return new CacheManager($app);
        });

        $this->app->singleton('cache.store', function ($app) {
            return $app['cache']->driver();
        });

        $this->app->singleton('memcached.connector', function () {
            return new MemcachedConnector;
        });
    }
```

到这一步可以看到Cache::get()实际上是调用了CacheManager类中的get方法。
```php
protected function get($name)
    {
        return $this->stores[$name] ?? $this->resolve($name);
    }
```
可以看到CacheManager中的get方法的属性是protected并不能从外部直接调用，所以猜测该类里应该有个__call()方法，果然：
```php
/**
     * Dynamically call the default driver instance.
     *
     * @param  string  $method
     * @param  array   $parameters
     * @return mixed
     */
    public function __call($method, $parameters)
    {
        return $this->store()->$method(...$parameters);
    }
```
在__call()中，首先调用了`$this->store()`:
```php
public function store($name = null)
    {
        $name = $name ?: $this->getDefaultDriver();

        return $this->stores[$name] = $this->get($name);
    }
```
`$this->getDefaultDriver()`方法代码:
```php
/**
     * Get the default cache driver name.
     *
     * @return string
     */
    public function getDefaultDriver()
    {
        return $this->app['config']['cache.default'];
    }
```
可以看到getDefaultDriver()方法的作用是获取默认缓存驱动。所以在`store()`方法中`$name`的作用就是保存要使用的缓存驱动。接着看`get()`方法。
```php
/**
     * Attempt to get the store from the local cache.
     *
     * @param  string  $name
     * @return \Illuminate\Contracts\Cache\Repository
     */
    protected function get($name)
    {
        return $this->stores[$name] ?? $this->resolve($name);
    }
```
可以看到`get()`方法是把`$this->resolve($name)`赋给`$this->stores[$name]`,接着看`$this->resolve($name)`:
```php
 /**
     * Resolve the given store.
     *
     * @param  string  $name
     * @return \Illuminate\Contracts\Cache\Repository
     *
     * @throws \InvalidArgumentException
     */
    protected function resolve($name)
    {
        $config = $this->getConfig($name);

        if (is_null($config)) {
            throw new InvalidArgumentException("Cache store [{$name}] is not defined.");
        }
        //是不是自定义的驱动
        if (isset($this->customCreators[$config['driver']])) {
            return $this->callCustomCreator($config);
        } else {
            //redis为例$driverMethod = 'createRedisDriver';
            $driverMethod = 'create'.ucfirst($config['driver']).'Driver';

            if (method_exists($this, $driverMethod)) {
                return $this->{$driverMethod}($config);
            } else {
                throw new InvalidArgumentException("Driver [{$config['driver']}] is not supported.");
            }
        }
    }
```

这里以redis为例，故`$name = 'redis'`,可以看出来`resolve()`最后返回的是`$this->createRedisDriver($config)`:

```php
protected function createRedisDriver(array $config)
    {
        $redis = $this->app['redis'];

        $connection = $config['connection'] ?? 'default';

        return $this->repository(new RedisStore($redis, $this->getPrefix($config), $connection));
    }
```
`$this->repository()`:
```php
/**
     * Create a new cache repository with the given implementation.
     *
     * @param  \Illuminate\Contracts\Cache\Store  $store
     * @return \Illuminate\Cache\Repository
     */
    public function repository(Store $store)
    {
        $repository = new Repository($store);

//        if ($this->app->bound(DispatcherContract::class)) {
//            $repository->setEventDispatcher(
//                $this->app[DispatcherContract::class]
//            );
//        }

        return $repository;
    }
```

最后返回了一个Repository类的实例，看下 class Repository类的构造方法：
```php
/**
     * Create a new cache repository instance.
     *
     * @param  \Illuminate\Contracts\Cache\Store  $store
     * @return void
     */
    public function __construct(Store $store)
    {
        $this->store = $store;
    }
```
在上面把Redis的操作对象 `new RedisStore($redis, $this->getPrefix($config)`放到了`$this->store`中，到这里可以看到`Cache::get()`方法实际上最终调用的是`class Repository 中的get()方法`:
```php

    /**
     * Retrieve an item from the cache by key.
     *
     * @param  string  $key
     * @param  mixed   $default
     * @return mixed
     */
    public function get($key, $default = null)
    {
        if (is_array($key)) {
            return $this->many($key);
        }
        //调用各个驱动里的get方法
        $value = $this->store->get($this->itemKey($key));

        // If we could not find the cache value, we will fire the missed event and get
        // the default value for this cache value. This default could be a callback
        // so we will execute the value function which will resolve it if needed.
        if (is_null($value)) {
            $this->event(new CacheMissed($key));

            $value = value($default);
        } else {
            $this->event(new CacheHit($key, $value));
        }

        return $value;
    }
```

至此`Cache::get()`整个过程结束。

