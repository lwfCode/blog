---
title: Go脚本实现模拟CPU+内存跑满程序
date: 2025-07-15 09:40:03
tags: Go
categories: 后端
top: 28
---

### 前言

在平时工作中，特别是在政务云环境下，客户经常觉得云服务器资源利用率过剩，所以经常写个小脚本模拟跑满cpu跟内存，因此记录一下。

#### 安装 Golang 环境

```
# 下载 Golang 安装包
wget https://golang.org/dl/go1.24.5.linux-amd64.tar.gz

# 解压安装包
tar -C /usr/local -xzf go1.24.5.linux-amd64.tar.gz

# 设置软链
ln -s /usr/local/go/bin/go /usr/bin/go

# 查看版本
go version
```

run.go

```
package main

import (
	"fmt"
	"math/rand"
	"runtime"
	"time"
)

const (
	targetCPUUsage     = 0.7 // CPU 占用目标
	durationPerCycle   = 100 * time.Millisecond
	memoryToAllocateGB = 10 // 目标内存：10GB
)

func burnCPU(done <-chan struct{}) {
	busyTime := time.Duration(float64(durationPerCycle) * targetCPUUsage)
	idleTime := durationPerCycle - busyTime

	for {
		select {
		case <-done:
			return
		default:
			start := time.Now()
			for time.Since(start) < busyTime {
			}
			time.Sleep(idleTime)
		}
	}
}

func burnMemory(targetGB int) [][]byte {
	fmt.Printf("开始分配 %d GB 内存...\n", targetGB)
	var blocks [][]byte
	for i := 0; i < targetGB*1024; i++ { // 每次分配 1MB，共 targetGB*1024 次
		b := make([]byte, 1024*1024)
		rand.Read(b) // 防止被优化
		blocks = append(blocks, b)
		if i%1024 == 0 {
			fmt.Printf("已分配 %d GB\n", i/1024)
		}
	}
	fmt.Println("内存分配完成")
	return blocks
}

func main() {
	numCPU := runtime.NumCPU()
	runtime.GOMAXPROCS(numCPU)

	done := make(chan struct{})

	for i := 0; i < numCPU; i++ {
		go burnCPU(done)
	}

	_ = burnMemory(memoryToAllocateGB)

	fmt.Println("资源模拟完成，程序退出")
	close(done)
}

```

#### 定时计划

```
#每周日的零点执行一次并记录日志
crontab -e

0 0 * * 7 /usr/local/bin/ go run /data/run.go > /data/run.log 2>&1
```

#### 最终效果如图

![alt text](/images/engin/666.jpg)