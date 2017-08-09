### axios简介：
axios 是一个基于Promise 用于浏览器和 nodejs 的 HTTP 请求库。

这个库本身具有的特征：

- 从浏览器中创建XMLHttpRequest
- 从node.js发出http请求
- 支持Promise
- 拦截请求和响应
- 转换请求和响应数据
- 取消请求
- 自动转换JSON数据
- 客户端方式 CSRF/XSRF

怎么使用就不讲了。既然有这么多特征，那么我们是不是就要来好好的了解下呢？是的，还是开始吧。

### 正文

还记得我们怎么使用这个`axios`呢？

```js
// node
import axios from 'axios'
axios.get('xxx')
    .then(res => {
        // ...  
    })
    .catch(err => {
        // ...
    })
// 浏览器
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
<script>
// 一。。样。。的。。
</script>
```
用起来是不是感觉美滋滋。超简单呢。
好吧，那这个时候我们就要去看看是在哪里`expose`出来的。看看入口在哪里。

>1、从lib文件夹下`axios.js`文件开始

开始引用了四个模块，先不关心这四个模块是干嘛的。

接下来是看：
```js

function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);

  // Copy context to instance
  utils.extend(instance, context);

  return instance;
}
```
主要是定义了一个创建了axios实例的函数。

接着就是以默认配置来创建一个axios的实例，并且将`Axios`类挂载在创建的实例上面以便于继承，接着当使用`axios.create({})`创建一个实例的时候将传进来的配置和默认的配置合并。
```js
// 默认配置创建实例
var axios = createInstance(defaults);
axios.Axios = Axios;
// 自定义配置创建实例
axios.create = function create(instanceConfig) {
  return createInstance(utils.merge(defaults, instanceConfig));
};
```
请求取消的相关操作暂时不读，接着就是对外暴露封装好的all方法以及封装的spread方法。这两个方法主要是用来控制并发请求的。最后就是以模块的当是导出，虽然这里是这样导出出去的，但是在打包的时候会支持多种加载方式的。
```js
axios.all = function all(promises) {
  return Promise.all(promises);
};
axios.spread = require('./helpers/spread');

module.exports = axios;
// Allow use of default import syntax in TypeScript
module.exports.default = axios;
```
### 最后
这个知识axios库的入口代码这块最核心的就是上面创建实例的方法。下一篇就是主要分析创建实例中的`Axios`这个“类”。