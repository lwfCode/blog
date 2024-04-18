---
title: Laravel+swoole实现异步任务
date: 2024-01-26 14:17:53
tags: Swoole
categories: Laravel
top: 10
---
## 前言：

这篇笔记记录了基于swoole的异步任务实现方式，框架采用的是laravel，当然框架环境只影响运行，不影响理解。

## 一、laravel添加commands

在项目根目录下生成一个command模块（因为swoole相关功能都是在cli模式下开发，可以配合相关进程管理工具来管理如：pm2，后续博文会讲到）

```
php artisan make:command Swoole/Task
```
然后在你的 console下的Kernel.php添加你刚刚生成的commands模块

目录： app\Console\Kernel.php

```
protected $commands = [
    Commands\Swoole\Task::class,
];
```

## 二、执行异步任务伪代码片段

```
<?php

namespace App\Console\Commands\Swoole;

use Illuminate\Console\Command;
use \Swoole\Http\Server;
use \Swoole\Coroutine as Go;
use \Swoole\Database\PDOConfig;
use \Swoole\Database\PDOPool;
use \Swoole\Runtime;
use Faker\Factory;

class Task extends Command
{
    /**
     * 这里是你启动服务的命令如：php artisan swoole:task
     * @var string
     */
    protected $signature = 'swoole:task';

    /**
     * 连接service
     * @var
     */
    private $service;

    /**
     * 任务处理成功返回code
     * @var int
     */
    public static $successCode = 200;

    /**
     * 服务host
     * @var string
     */
    private $host = "0.0.0.0";

    /**
     * 端口号
     * ps:linux环境下阿里云服务器自行开放安全组端口
     * 防火墙开放相关端口等
     * @var int
     */
    private $port = 9502; 

    /**
     * 任务处理失败返回code
     * @var int
     */
    public static $errorCode = 0;

    /**
     * 参数为空返回code
     * @var int
     */
    public static $paramsCode = 400;


   
    public function __construct()
    {
        parent::__construct();
        //$this->facker = Factory::create('zh_CN');
    }


    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        return $this->init();
    }


    /**
     * 初始化server
     * return void
     */
    private function init()
    {
        $this->service = new Server($this->host,$this->port);
        $this->service->set([
            'task_worker_num' => 4 //开启4个worker进程
        ]);
        /**
        进程数设置需要考虑以下条件
        1、cpu核数
        2、内存大小
        3、业务偏向IO密集还是CPU密集型
        如果业务代码偏向IO密集型，也就是业务代码有IO阻塞的地方，则根据IO密集程度设置进程数，例如CPU核数的3倍。
        如果业务代码偏向CPU密集型，也就是业务代码中无IO通讯或者无阻塞式IO通讯，则可以将进程数设置成cpu核数
        
        注意：swoole异步客户端的IO都是非阻塞的，属于CPU密集型操作。如果不清楚自己业务偏向于哪种类型，可设置进程数为CPU核数的2倍左右即可
        */

        $this->service->on("request", [$this, 'onRequest']);
        $this->service->on("task", [$this, 'onTask']);
        $this->service->on("finish", [$this, 'onFinish']);

        $this->service->start();
    }

    /**
     * 任务请求接受
     * @param $request
     * @param $response
     * @return mixed
     */
    public function onRequest($request, $response)
    {
        if(strpos($request->server['request_uri'],'.ico') !== false){
            $response->end(self::$errorCode);
        }else{
            $data = $request->post;
            if(!isset($data)){
                return $this->resultSuccess($response,self::$paramsCode);
            }
            $this->service->task($data);
            $response->end(self::$successCode);
        }
    }


    /**
     * 处理任务
     * @param \Swoole\Server $server
     * @param $task_id
     * @param $from_id
     * @param $data
     */
    public function onTask(\Swoole\Server $server, $task_id, $from_id, $data)
    {
        echo "异步任务Data处理中:".json_encode($data).PHP_EOL;
        sleep(2); //这里设置一下2秒sleep方便测试
        $server->finish($data);
    }


    /**
     * task进程任务处理结束回调，可选
     * @param \Swoole\Server $server
     * @param $task_id
     * @param $data
     * @return mixed
     */
    public function onFinish(\Swoole\Server $server, $task_id, $data)
    {
        echo "异步任务结束,数据:".json_encode($data).PHP_EOL;
        $result = ['data' => $data];
        return Result(200,'success',$result);
    }


    /**
     * 返回函数
     * return string
     */
    protected function resultSuccess($response,$code = 0)
    {
        return $response->end($code);
    }
}

```

## 三、启动异步任务测试

1.开启终端输入：

    php artisan swoole:task

我们通过postman代替来访问，因为我swoole服务端起的是 http服务，其他如tcp服务，你可以通过swoole的客户端来连接

##### 连续访问2次，每次task\_id id不一样

此时我们可以开终端打印的数据,已经做到了异步处理任务了。


#### 最后补充一点，有小伙伴咨询为什么我不是通过 ip+端口号方式访问，因为在nginx那里做了反向代理

