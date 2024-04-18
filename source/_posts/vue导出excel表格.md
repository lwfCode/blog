---
title: vue导出excel表格
date: 2023-01-26 14:23:21
tags: vue
categories: 前端
top: 2
---


在开发过程中有一个项目需求，要求在前端项目中导出Excel表格，经查,Vue.js确实可以实现，具体实现步骤为：

#### 1.首先安装依赖包

没有安装node请先安装：[node传送](http://nodejs.cn)

```
//npm 
npm install -S file-saver xlsx
npm install -D script-loader

或者

//yarn
yarn add file-saver
yarn add xlsx
yarn add script-loader --dev
```

#### 2.导入js插件

[下载Blob.js和Export2Excel.js](https://download.csdn.net/download/dt1991524/10911275)，在自己项目目录（一般是src）下新建excel目录，我这里是src/layouts/local/excel，直接放入即可

在main.js中引入
```

***代表路径文件名称

import Blob from './***/Blob'
import Export2Excel from './***/Export2Excel.js'

```


然后在你需要导出excel表格的vue文件中使用

tHeader是表头，filterVal 中的数据是表格的字段，data中存放表格里的数据，类型为数组，里面存放对象，表格的每一行为一个对象。

如：Stration.vue


```
 //请求后端获取数据
 async getLoadList(){
         this.loading =  true
         const { body: { list } }  = await this.$request.post(api => api.local.station,{
          user_token: this.userToken,
          getExcel: 1
      })
      return list
    },
   async getExcel() {
     let result = await this.getLoadList()
     this.loading = false
     //导出excel
     require.ensure([], () => {
            const { export_json_to_excel } = require('./excel/Export2Excel.js') //请注意对应路径
            const tHeader = ['编号', '用户名称','手机号码','审核状态','报名时间']
            const filterVal = ['id', 'realname','mobile','state','created_time']
            const list = result
            list.map(item => {
               //item['job'] = this.CaptionUser(item['job'])
                if (item['state'] == 0){
                    item['state'] = '未审核';
                }else if ($item['state'] == 1){
                    item['state'] = '审核通过';
                }else{
                    item['state'] = '拒绝';
                }
            })
            const data = this.formatJson(filterVal, list)
            export_json_to_excel(tHeader, data, 'xxx列表')
        })
    },
    formatJson(filterVal, jsonData) {
        return jsonData.map(v => filterVal.map(j => v[j]))
    },
```

#### 三.当然你也可以封装成一个公共函数全局调用，这样可能更加方便

新建一个js文件: src/plugins/global_variable/index.js


```
let tools = {}
tools.ExcelData = function(params = {}) {
  /**
   * 导出Excel表格
   * @header  表头单元格
   * @filter  表格数据对应形参
   * @result  导出数据
   * @title   表格标题
   *  */
  require.ensure([], () => {
      const { export_json_to_excel } = require('@/layouts/local/excel/Export2Excel.js')
      const tHeader = params.header
      const filterVal = params.filter
      const list = params.result
      const data = formatJson(filterVal, list)
      export_json_to_excel(tHeader, data, params.title)
  })
}

function formatJson(filterVal, jsonData) {
  return jsonData.map(v => filterVal.map(j => v[j]))
}

tools.install = function (Vue, options) {
  Vue.prototype.$tools = tools
  Vue.tools = tools
}
export default tools
```

然后在main.js中挂载在vue实例中

```
import tools from './plugins/global_variable/index'//引用
Vue.use(tools)
```


使用

```
async getList(){
     this.loading =  true
     const { body: { list } }  = await this.$request.post(api => api.publicity.excel,{
      user_token: this.userToken,
  })
  return list
},
async Excel(){
  if(this.name == 'gongyuxing'){
    let result = await this.getList()
    this.loading = false
      let body = {
         result,
         header:['编号', '申请人姓名'],
         filter:['id', 'realname'],
         title: '****列表'
       }
      return this.$tools.ExcelData({...body})//全局调用
  }else{
    return this.$message({
        message: 'Sorry,专用!',
        type: 'error'
      })
  }
}
```

---

ok，完毕！