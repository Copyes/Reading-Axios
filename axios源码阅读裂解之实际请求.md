### 实际请求

我们在前面Axios的构造函数那边有看的将请求拦截，“实际请求”，响应拦截组成调用链的。实际请求是怎么来实现的呢？

这个“实际请求”呢?

>1、dispatchRequest.js

进到这个文件发现下面这个函数：
```js
function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}
```
这个函数主要的作用就是判断是有通过CancelToken来消除请求的。这个token是通过使用CancelToken.source工厂创建的一个取消令牌。因为这个操作我们不常用，所以不作过多的解释。

接着看接下来导出出去的`dispatchRequest`方法：

```js
module.exports = function dispatchRequest(config) {
    //...
}
```
这个方法里面进行的操作是：
-   1、判断请求头存在否
-   2、转换请求数据
-   3、展开headers

```js
config.headers = config.headers || {};

// Transform request data
config.data = transformData(
config.data,
config.headers,
// 用户自己设置的转换函数，将headers作为第二个参。
// 这样可以根据headers里面的信息去做不同的操作。
config.transformRequest
);

// Flatten headers
config.headers = utils.merge(
config.headers.common || {},
config.headers[config.method] || {},
config.headers || {}
);

utils.forEach(
['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
function cleanHeaderConfig(method) {
  delete config.headers[method];
}
);
```
执行完上面的操作后，接下来就是下面:
这里主要做的操作是调用请求，并收到响应后处理响应数据。这里需要返回的是一个promise。

```js
var adapter = config.adapter || defaults.adapter;

return adapter(config).then(function onAdapterResolution(response) {
    throwIfCancellationRequested(config);

    // 转换响应数据
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);

      // Transform response data
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }

    return Promise.reject(reason);
});
```