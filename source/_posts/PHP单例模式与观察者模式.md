---
title: PHP单例模式与观察者模式
date: 2024-04-15 13:27:43
tags: PHP
categories: 后端
top: 18
---

### 一、观察者模式

当一个对象的状态发生改变时，依赖他的对象会全部收到通知，并自动更新。

##### 场景：一个事件发生后，要执行一连串更新操作。传统的编程方式，就是在事件的代码之后直接加入处理逻辑，当更新的逻辑增多之后，代码会变得难以维护。这种方式是耦合的，侵入式的，增加新的逻辑需要改变事件主题的代码
观察者模式实现了低耦合，非侵入式的通知与更新机制，例如 Laravel 的事件就提供了一个简单的观察者实现，能够订阅和监听应用中发生的各种事件。


```

<?php

/**
 * 观察者接口类
 * Interface ObServer
 */
interface ObServer
{
    public function update($event_info = null);
}

/**
 * 观察者1
 */
class ObServer1 implements ObServer
{
    public function update($event_info = null)
    {
        echo "观察者1 收到执行通知 执行完毕！\n";
    }
}

/**
 * 观察者2
 */
class ObServer2 implements ObServer
{
    public function update($event_info = null)
    {
        echo "观察者2 收到执行通知 执行完毕！\n";
    }
}

/**
 * 事件
 * Class Event
 */
class Event
{
    //增加观察者
    public function add(ObServer $ObServer)
    {
        $this->ObServers[] = $ObServer;
    }
    //事件通知
    public function notify()
    {
        foreach ($this->ObServers as $ObServer) {
            $ObServer->update();
        }
    }
    /**
     * 触发事件
     */
    public function trigger()
    {
        //通知观察者
        $this->notify();
    }
}

//创建一个事件
$event = new Event();

//为事件增加旁观者
$event->add(new ObServer1());
$event->add(new ObServer2());

//执行事件 通知旁观者
$event->trigger();

```


### 一、单例模式

这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

<mark>注意：</mark>
1、单例类只能有一个实例。
2、单例类必须自己创建自己的唯一实例。
3、单例类必须给所有其他对象提供这一实例。


```
<?php

clasee Singleton 
{
    
    private static $instance;
    
    //私有构造方法，禁止使用new创建对象
    private function __construct(){}
    
    public static function getInstance(){
        
        if (!isset(self::$instance)){
            self::$instance = new self;
        }
        return self::$instance;
    }
    
    //将克隆方法设置为私有，禁止克隆对象
    private function __clone(){}
    
    public function debug($name = '测试'){
        
        echo "这里是单例模式创建对象实例<br>".$name;
    }
}
```

测试：


```
$sing = Singleton::getInstance();
$sing->debug("hello")

$newSing = Singleton::getInstance();
var_dump($sing === $newSing);
```
