## 前言

> - Hexo 是一款基于 `Node.js` 的静态博客框架，依赖少易于安装使用，同时拥有着众多插件和主题，支持 Git、云主机、对象存储等方式部署发布。

## Hexo 快速入门

Hexo 的使用和调试 基于 `Node.js`，通过 `npm install -g hexo && npm install -g hexo-cli` 可以将 Hexo 全局安装到系统中调用。也可以通过 `npm install hexo --save` 将 Hexo 集成在当前的项目中。相关 Hexo 命令学习可以通过访问 [Hexo快速入门] 进行学习了解

Hexo 使用 Markdown 语法进行编写，支持常见的 [Markdown](https://daringfireball.net/projects/markdown/) 语法。


### 流水线配置结构和字段说明

```yaml
# ========================================================
# 基于 aliyun OSS / tencent COS 构建部署 Hexo 静态网站示例
# 功能：通过 Node 编译构建 Hexo 项目并部署到 OSS / COS
# ========================================================
name: hexo-oss-deploy                      # 定义一个唯一 ID 标识为 hexo-oss-deploy ，名称为「 Hexo 部署(OSS) 」的流水线
displayName: 'Hexo 部署(OSS)'
triggers:                                  # 流水线触发器配置
  push:                                    # 设置 master 分支 在产生代码 push 时精确触发（PRECISE）构建
    - matchType: PRECISE
      branch: master
commitMessage: ''                          # 通过匹配当前提交的 CommitMessage 决定是否执行流水线
stages:                                    # 构建阶段配置
  - stage:                                 # 定义一个 ID 标识为 deploy-stage ,名为「 Deploy Stage 」的阶段
      name: deploy-stage
      displayName: 'Deploy Stage'
      failFast: false                      # 允许快速失败，即当 Stage 中有任务失败时，直接结束整个 Stage
      
      steps:                               # 构建步骤配置
        - step: npmbuild@1                 # 采用 npm 编译环境
          name: deploy-step                # 定义一个 ID 标识为 deploy-step ,名为「 Deploy Step 」的阶段
          displayName: 'Deploy Step'
          inputs:                          # 构建输入参数设定
            nodeVersion: 14.15             # 指定 node 环境版本为 14.15
            goals: |                       # 安装依赖，配置相关主题、部署参数并发布部署
              node -v
              npm -v
              npm install
              npm run config url $SITE
              npm run config theme $THEME
              npm run config deploy.cloud $CLOUD
              npm run config deploy.bucket $BUCKET
              npm run config deploy.region $REGION
              npm run config deploy.secretId $SECRET_ID
              npm run config deploy.secretKey $SECRET_KEY
              npm run clean
              npm run deploy
```

---

[Gitee Go]:https://gitee.com/features/gitee-go#production-examples
[Hexo快速入门]:./tutorials/hexo-quick-start.md

### demo address
```
http://blog.yikuaida.cn
```
