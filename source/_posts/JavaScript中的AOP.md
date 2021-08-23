---
title: JavaScript的函数式编程(-)
date: 2021-08-23 11:09:59
tags: javascript 函数式编程 currying 高阶函数
toc: true
---

## 使用函数式编程实现接口缓存

假设我们需要请求一个接口

```javascript
const get = (url, params) => Promise.resolve(params);

const getData = (params) => get("/api", params);

const appendDiv = (text) => {
  const div = document.createElement("div");
  div.innerText = text;
  document.body.appendChild(div);
};

getData({ q: 123 }).then((data) => {
  appendDiv(JSON.stringify(data));
});
```

## 使用缓存优化性能

现在为了性能优化,我们需要对性能优化需要做缓存处理

```javascript
// ...

const cacheMap = {};

const getKey = (url, params) =>
  JSON.stringify({
    __url: url,
    ...params,
  });

const params = { q: 123 };
const key = getKey("/api", params);

const handleData = (data) => appendDiv(JSON.stringify(data));

if (cacheMap[key]) {
  handleData(cacheMap[key]);
} else {
  getData(params).then((data) => {
    cacheMap[key] = data;
    handleData(data);
  });
}
```

## 利用代理模式分离缓存和业务逻辑

我们可以利用代理模式的思路,将缓存和请求分离开来

```javascript
const getDataProxy = async (params) => {
  const key = getKey("/api", params);
  if (cacheMap[key]) {
    return cacheMap[key];
  }
  return getData(params);
};

const handleDataProxy = (params, data) => {
  const key = getKey("/api", params);
  cacheMap[key] = data;
  handleData(data);
};

getDataProxy(params).then((data) => handleDataProxy(params, data));
```

## 使用 currying 和高阶函数复用代码

虽然我们现在把当前请求的缓存和请求分离开来,但是只限定当前请求,并不能通用化
我们使用高阶函数来实现代码复用

```javascript
import { compose, tap, curry } from "lodash/fp";

//...
const ifElse = (flag, firstFun, secondFun) => {
  return flag ? firstFun : secondFun;
};

const getCache = (key) => cacheMap[key];

const setCache = curry((key, data) => (cacheMap[key] = data));

const proxyRequest = curry((url, request, params) => {
  const key = getKey(url, params);
  return ifElse(getCache(key), () => getCache(key), request)(params);
});

const proxyHandleData = curry((url, handleData, params, data) => {
  const key = getKey(url, params);
  setCache(key, data);
  handleData(data);
});

const getDataProxy = proxyRequest("/api", getData);
const handleDataProxy = proxyHandleData("/api", handleData);

getDataProxy(params).then(handleDataProxy(params));
```
