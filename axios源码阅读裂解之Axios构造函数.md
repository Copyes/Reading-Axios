### Axios类的实现

Axios类的实现是比较简单的，就是将传进来的配置赋值给默认配置，然后通过`this.interceptors`对请求拦截器和响应拦截器进行管理。
```js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
```
接着直接看看InterceptorManager构造函数做了什么。
```js
// 构造函数自身只定义了一个用于存各个拦截器的回调的数组。
// 当使用一个拦截器的时候就想`handlers`中添加一个回调对象数组,这个对象就包含的是promise中resolve和reject的回调
function InterceptorManager() {
  this.handlers = [];
}
// 拦截器使用的时候传的是两个参数
// 一个是作为promise的resolve()的回调
// 一个是作为promise的reject()的回调
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};
// 使用id来辨别是哪个拦截器，然后就是用id来去除一个拦截器
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};
// 遍历所有注册了的拦截器。
// 这个方法会跳过已经被设置为null的拦截器，遍历执行fn
// 值得一提的是，移除方法是通过直接将该项设为null实现的，而不是用splice剪切该数组，遍历方法中也增加了相应的null值处理。
// 这样做一方面使得每一项ID保持为项的数组索引不变，另一方面也避免了重新剪切拼接数组的性能损失
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};
```
看完了拦截器管理构造函数做了什么操作，接着往后看Axios构造函数。 这里的操作就是处理合并请求的配置信息相关，比如增加请求method相关，支持默认baseURL的配置。

```js

Axios.prototype.request = function request(config) {
  // 当使用方式是 axios('example/url'[, config]) 这种的时候，将配置整理合并
  if (typeof config === 'string') {
    config = utils.merge({
      url: arguments[0]
    }, arguments[1]);
  }

  config = utils.merge(defaults, this.defaults, { method: 'get' }, config);
  config.method = config.method.toLowerCase();

  // Support baseURL config
  if (config.baseURL && !isAbsoluteURL(config.url)) {
    config.url = combineURLs(config.baseURL, config.url);
  }
  //...
  
};
```
从这里开始就比较巧妙了。实际请求，请求拦截，响应拦截，通过chain这个数组变量来存储和管理了。
就是经过处理后变成：
```js
［请求拦截1，请求拦截2，...，实际请求，undefined，响应拦截1，响应拦截2，...］

```
最终依次通过then方法添加到promise中去。
```js
xxx{
    // ...
    var chain = [dispatchRequest, undefined];
    var promise = Promise.resolve(config);
    
    this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
    });
    
    this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
    });
    
    while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
    }
    
    return promise;
}
```
最后相当于执行的是如下代码：
```js
promise
    .then(request[1].fulfilled,request[1].rejected)
    .then(request[2].fulfilled,request[2].rejected)
    .then(dispatchRequest,undefined)
    .then(response[1].fulfilled,response[1].rejected)
    .then(response[1].fulfilled,response[1].rejected)
```
继续执行就到了将对应的http方法转换成请求的时候了。具体代码就是下面：
```js
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  Axios.prototype[method] = function(url, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  Axios.prototype[method] = function(url, data, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});
```
### 总结
Axios构造函数看起来实现起来比较简单，但是在处理请求链路那一点还是比较好的，巧妙的利用的unshift、push、shift等数组队列、栈方法来实现了请求拦截、执行请求、响应拦截的流程设定。
