---
title: PHP+Swoole建立WebSocket连接实时推送消息
date: 2025-12-18 13:59:55
tags: WebSocket
categories: 后端
top: 20
---


##### 步骤 1：生成命令类骨架（TP6 内置命令）

```
php think make:command websocket/WebSocket
```

执行后会提示：Command:app\command\websocket\WebSocket created successfully

##### 步骤 2：替换生成的骨架代码（适配 WebSocket）

```
<?php
declare(strict_types=1);

namespace app\command\websocket;

use Swoole\Process;
use think\console\{Command, Input, input\Argument, Output};
use Swoole\WebSocket\{Server,Frame};
use Swoole\{Table,Timer};

class WebSocket extends Command
{
    protected $server;
    private $host = '0.0.0.0';
    private $port = 9503;

    /**
     * 1. 存储 FD 状态（在线/离线）
     * 2. 同时存储 FD 对应的 params 参数
     */
    private $fdTable;
    private $pidFile;
    private $MachineService;

    /** 每200ms 推送一次 */
    const TIMER_INTERVAL = 200;
    const PARAMS_COLUMN_LEN = 4096;
    const ERROR_EXPIRE_TIME = 300; // 异常标记过期时间（秒），300秒后自动重置
    const SIGNAL_RELOAD = SIGUSR1; // 平滑重启信号


    protected function configure()
    {
        $this->setName('websocket')
            ->setDescription('WebSocket Server')
            ->addArgument("action",Argument::REQUIRED,"服务器操作：start|stop|reload|status");
    }

    protected function execute(Input $input, Output $output)
    {
        // 初始化PID文件路径（TP6 runtime目录，避免权限问题）
        \think\facade\Env::set('runtime_path', root_path() . 'runtime/');
        $this->pidFile = \think\facade\Env::get('runtime_path') . '/websocket.pid';

        $action = $input->getArgument('action');
        switch ($action) {
            case 'start':
                $this->startServer($output);
                break;
            case 'stop':
                $this->stopServer($output);
                break;
            case 'restart':
                $this->restartServer($output);
                break;
            case 'reload':
                $this->reloadServer($output);
                break;
            case 'status':
                $this->checkStatus($output);
                break;
            default:
                $output->error('无效操作！支持：start|stop|reload|status');
                exit(1);
        }
    }



    /**
     * 强制重启：先停止（强制杀进程+删PID），再启动
     */
    private function restartServer(Output $output)
    {
        // 1. 强制停止旧进程（不管是否正常运行）
        $this->forceStopServer($output);
        // 2. 延迟2秒，确保端口释放
        sleep(2);
        // 3. 启动新进程
        $this->startServer($output);
    }

    /**
     * 强制停止：杀所有相关进程 + 删除PID文件
     */
    private function forceStopServer(Output $output)
    {
        // 杀死所有websocket相关进程
        exec("ps aux | grep 'php think websocket' | grep -v grep | awk '{print $2}' | xargs kill -9 2>/dev/null");
        // 删除PID文件
        if (file_exists($this->pidFile)) {
            unlink($this->pidFile);
        }
        $output->info("所有WebSocket进程已强制关闭！");
    }


    public function startServer(Output $output)
    {
        try {

            // 初始化 Swoole\Table：维护 FD 列表
            $this->initFdTable();

            $this->server = new Server($this->host, $this->port);
       
            $this->server->set([
                'worker_num' => swoole_cpu_num(), // 按CPU核数设置Worker数
                'daemonize' => true,//守护进程启动
                'max_request' => 100,
                'reload_async' => true, // 异步重启（平滑重启核心配置）
                'max_wait_time' => 2, // 旧Worker等待时间（秒），确保连接处理完
                'pid_file' => $this->pidFile,
            ]);

            $this->server->on("Start", [$this, 'onStart']);
            $this->server->on("Shutdown", [$this, 'onShutdown']);
            $this->server->on("workerStart", [$this, 'onWorkerStart']);
            $this->server->on("open", [$this, 'onOpen']);
            $this->server->on("message", [$this, 'onMessage']);
            $this->server->on("close", [$this, 'onClose']);

            $this->server->start();
        } catch (\Exception $e) {
            $output->error("启动失败：{$e->getMessage()}");
            exit(1);
        }
    }

    /**
     * 停止服务（强制关闭）
     */
    private function stopServer(Output $output)
    {
        if (!$this->isServerRunning()) {
            $output->error("WebSocket服务未运行！");
            exit(1);
        }

        $pid = $this->getPid();
        // 发送停止信号（SIGTERM）
        if (Process::kill($pid, SIGTERM)) {
            // 等待进程退出
            usleep(500000);
            // 删除PID文件
            if (file_exists($this->pidFile)) {
                unlink($this->pidFile);
            }
            $output->info("WebSocket服务已停止！PID：{$pid}");
        } else {
            $output->error("停止服务失败！PID：{$pid}");
            exit(1);
        }
    }

    /**
     * 平滑重启服务（仅重启Worker进程，不中断连接）
     */
    private function reloadServer(Output $output)
    {
        if (!$this->isServerRunning()) {
            $output->error("WebSocket服务未运行！");
            exit(1);
        }

        $pid = $this->getPid();
        // 发送平滑重启信号（SIGUSR1）
        if (Process::kill($pid, self::SIGNAL_RELOAD)) {
            $output->info("WebSocket服务平滑重启中... PID：{$pid}");
        } else {
            $output->error("平滑重启失败！PID：{$pid}");
            exit(1);
        }
    }

    /**
     * 检查服务状态
     */
    private function checkStatus(Output $output)
    {
        if ($this->isServerRunning()) {
            $output->info("WebSocket服务运行中！PID：{$this->getPid()}，地址：{$this->host}:{$this->port}");
        } else {
            $output->info("WebSocket服务已停止！");
        }
    }

    /**
     * 初始化 FD 管理表：
     * - key：FD 字符串（Swoole\Table 的 key 仅支持字符串）
     * - columns：is_online（是否在线）、params（参数）、connect_time（连接时间）
     */
    private function initFdTable()
    {
        $this->fdTable = new Table(10240);
        $this->fdTable->column('is_online', \Swoole\Table::TYPE_INT, 1);
        $this->fdTable->column('params', \Swoole\Table::TYPE_STRING, self::PARAMS_COLUMN_LEN);
        $this->fdTable->column('connect_time',\Swoole\Table::TYPE_INT, 8);
        $this->fdTable->column('error_flag', \Swoole\Table::TYPE_INT, 1);
        $this->fdTable->column('error_msg', \Swoole\Table::TYPE_STRING, 1024);
        $this->fdTable->column('error_time', \Swoole\Table::TYPE_INT, 8); // 异常发生时间
        $this->fdTable->create();

    }

    /**
     * 主进程启动事件：注册信号监听
     */
    public function onStart(Server $server)
    {
        echo "webscket服务器已启动";
    }


    /**
     * 服务关闭事件
     */
    public function onShutdown(Server $server)
    {
        echo "WebSocket服务已关闭！\n";
        // 清理PID文件
        if (file_exists($this->pidFile)) {
            unlink($this->pidFile);
        }
    }


    public function onWorkerStart(Server $server, int $workerId)
    {
        //你的业务代码（实例化类） 
        //清理opcache（若有）
//        if (function_exists('opcache_reset')) {
//            opcache_reset();
//        }

        //定时推送
        if ($workerId === 0) {
            Timer::tick(self::TIMER_INTERVAL, function (int $timerId) use ($server) {
                $this->timerPush($server); // 定时点对点推送
            });
        }
    }

    /**
     * 检查服务是否运行
     */
    private function isServerRunning(): bool
    {
        $pid = $this->getPid();
        if (!$pid) {
            return false;
        }
        // 检查PID是否有效
        return Process::kill($pid, 0);
    }


    /**
     * 从PID文件获取主进程PID
     */
    private function getPid(): ?int
    {
        if (!file_exists($this->pidFile)) {
            return null;
        }
        $pid = (int)file_get_contents($this->pidFile);
        return $pid > 0 ? $pid : null;
    }

    /**
     * 客户端连接：记录 FD 到 Swoole\Table
     */
    public function onOpen(Server $server, \Swoole\Http\Request $request)
    {
        $fd = $request->fd;

        echo "客户端连接 FD：{$fd}\n";
        // 存储 FD 状态和初始参数
        $this->fdTable->set((string)$fd, [
            'is_online' => 1,
            'params' => json_encode($request->post ?? [], JSON_UNESCAPED_UNICODE),
            'connect_time' => time(),
            'worker_exit_timeout' => 1, // 新增：Worker退出超时时间
            'error_flag' => 0, // 初始无异常
            'error_msg' => '',
            'error_time' => 0
        ]);
    }

    /**
     * 接收消息：更新 FD 参数
     */
    public function onMessage(Server $server, Frame $frame)
    {
        $fd = $frame->fd;
        $data = json_decode($frame->data, true) ?: [];

        if (!empty($data)) {
            $oldRow = $this->fdTable->get((string)$fd);

            if (isset($oldRow['error_flag']) && $oldRow['error_flag'] === 1) {
                $this->fdTable->set((string)$fd, [
                        'error_flag' => 0,
                        'error_msg' => '',
                        'error_time' => 0
                    ] + $oldRow);
                echo "FD:{$fd} 异常标记已重置\n";
            }

            $newParams = $data;
            $this->fdTable->set((string)$fd, [
                'is_online' => 1,
                'params' => json_encode($newParams, JSON_UNESCAPED_UNICODE),
                'connect_time' => $oldRow['connect_time'] ?? time()
            ]);
            //推送消息
            $this->timerPush($server);
        }
    }

    /**
     * 客户端断开：标记 FD 为离线
     */
    public function onClose(Server $server, int $fd)
    {
        echo "客户端断开 FD：{$fd}\n";
        // 标记离线（根据是否需要保留历史）
//        $this->fdTable->set((string)$fd, [
//            'is_online' => 0,
//            'params' => $this->fdTable->get((string)$fd)['params'],
//            'connect_time' => $this->fdTable->get((string)$fd)['connect_time']
//        ]);
        // 直接删除 FD 记录（无需保留历史时）
        $this->fdTable->del((string)$fd);
    }

    /**
     * 定时点对点推送：遍历 Swoole\Table 中的在线 FD
     */
    private function timerPush(Server $server)
    {

        foreach ($this->fdTable as $fdStr => $row) {
            $fd = (int)$fdStr;

            if ($row['is_online'] !== 1 || !$server->isEstablished($fd)) {
                // 清理无效 FD
               //$this->fdTable->set($fdStr, ['is_online' => 0] + $row);
                $this->fdTable->del((string)$fdStr);
                continue;
            }

            // 校验异常标记：已推送过则跳过
            if ($row['error_flag'] === 1) {
                // 异常标记过期后自动重置
//                if (time() - $row['error_time'] > self::ERROR_EXPIRE_TIME) {
//                    $this->fdTable->set($fdStr, [
//                            'error_flag' => 0,
//                            'error_msg' => '',
//                            'error_time' => 0
//                        ] + $row);
//                }
                continue;
            }

            $params = json_decode($row['params'], true) ?: [];

            try {
                //业务代码
                if (!empty($params) && $params['type'] === "debug"){
                    var_dump("debug...");
                }
            }catch (\Exception $exception){
                $errorData = [
                    'code' => 0,
                    'msg' => $exception->getMessage(),
                    'data' => []
                ];
                $server->push($fd, json_encode($errorData, JSON_UNESCAPED_UNICODE));
                //设置异常标识，仅推送一次异常信息
                $this->fdTable->set($fdStr, [
                        'error_flag' => 1,
                        'error_msg' =>  $exception->getMessage(),
                        'error_time' => time()
                    ] + $row);
            }
        }
    }
}
```

##### 步骤 3：服务端运行

![alt text](/images/websocket/websocket.png)