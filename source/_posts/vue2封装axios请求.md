---
title: vue2封装axios请求
date: 2023-01-28 11:04:18
tags: vue
categories: 前端
top: 10
---

### 一.为什么要封装axios请求

在实际开发中基本上都是以前后端分离为主，在后台交互获取数据这块，通常使用的是axios库，它是基于promise的http库，可运行在浏览器端和node.js中。他有很多优秀的特性，例如拦截请求和响应、取消请求、转换json、客户端防御CSRF等

### 二.安装axios

```
npm install axios;
```
一般我会在项目的src目录中，新建一个request文件夹，然后在里面新建一个request.js和一个api.js文件。request.js文件用来封装我们的axios，api.js用来统一管理我们的接口。

在request.js中引入axios

```
import Vue from 'vue'
import axios from 'axios'; // 引入axios
import store from '@/store'
```

### 三.环境的切换
我们的项目环境可能有开发环境、测试环境和生产环境。我们通过node的环境变量来匹配我们的默认的接口url前缀。

axios.defaults.baseURL可以设置axios的默认请求地址就不多说了

```
let _getWholeUrl = (action, base) => {
  let baseUrl = base
  if (base === undefined) {
    let env = process.env
    if (env.NODE_ENV === 'development') {
      baseUrl = 'http://test-api.cn'
    } else if (env.NODE_ENV === 'production') {
      baseUrl = 'https://api.cn'
    }
  }
  return baseUrl + action
}


let _responseHook = (response) => {
  console.log(response)
  if (response.headers.has('X-SMS-Session')) {
    // 存入store
    store.commit('setSMSSession', response.headers.get('X-SMS-Session'))
  }
  return new Promise((resolve) => {
    resolve(response)
  })
}

let _errorHook = (error) => {
  window.alert('系统错误，请稍后再试')
  store.commit('Loading', false)
  return new Promise((resolve, reject) => {
    reject(error)
  })
}

```

### 四.封装get，post,delete,put等方法


```
/**
 * 封装方法
 * @param url
 * @param data
 * @returns {Promise}
 */
let _post = (url, param = {}) => {
  if (typeof url === 'function') {
    url = url(Vue.api)
  }
  return axios.post(_getWholeUrl(url), param,{ emulateJSON: true })
}

let _get = (url, param = {}) => {
  if (typeof url === 'function') {
    url = url(Vue.api)
  }
  return axios.get(_getWholeUrl(url), {
    params: param
  })
}

let _delete = (url, param = {}) => {
  if (typeof url === 'function') {
    url = url(Vue.api)
  }
  return axios.delete(_getWholeUrl(url), {
    params: param
  })
}
let _put = (url, param = {}) => {
  if (typeof url === 'function') {
    url = url(Vue.api)
  }
  return axios.put(_getWholeUrl(url), param,{ emulateJSON: true })
}
```
#### 五.导出当前封装的请求


```
export default {
  post: _post,
  get: _get,
  url: _getWholeUrl,
  delete: _delete,
  put: _put
}
```

#### 六：在request下新建index.js,api.js

request/index.js


```
/**
 * http访问
 */
import API from './api'
import Request from './request'

function plugin (Vue) {
  if (plugin.installed) {
    return
  }

  Vue.request = Request
  Vue.api = API
  Object.defineProperties(Vue.prototype, {
    $request: {
      get () {
        return Request
      }
    },
    $api: {
      get () {
        return API
      }
    }
  })
}

export default plugin
```

request/api.js



```
export default {
    user: {
        login: '/admin/user/login'
    },
    index:{
       index: '/index'
    }
}
```


#### 七：在main.js中引入挂载到vue实例中


```
import Vue from 'vue'
import App from './App'
import router from './router'
import store from './store'
import Request from './request' //引入
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'

Vue.config.productionTip = false
Vue.use(Request)
Vue.use(ElementUI)

Vue.directive('title', {
  inserted: function (el) {
    document.title = el.innerText + ' - Mr.lei'
    el.remove()
  }
})

/* eslint-disable no-new */
new Vue({
  el: '#app',
  store,
  router,
  template: '<App/>',
  // components: { App }
  render: h => h(App)
})

```
#### 八：在项目中使用


```
this.$request.post(api => api.index.index, {
    user_token: this.userToken,
  }).then(response => {
    console.log(response)
})
```