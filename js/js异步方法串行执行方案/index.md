
---
title: Win11下安装docker
subtitle: 异步方法串行执行方案
date: 2023年7月26日
tags: 

  - js
categories:
  - js

---


![img](assets/promises.png)

在封装第三方库的异步方法场景中，需要保证系统中该第三方方法是串行化执行的。

<!--more-->

## 场景描述

由于第三方接口调用频率要求，同一时刻只能有一个方法正在执行，不能有并发操作。在使用第三方接口时存在统一封装，封装后对外提供，会有多处功能使用。



## 方案

基于`Promise`执行状态，及`async`中`await`等待执行机制，实现如下：

```js
let lock_promise = undefined;
async function serialPack(task) {
    if (lock_promise) {
        await lock_promise;
        //锁释放，再去竞争锁
        console.log('竞争锁')
        return serialPack(task);
    } else {
        //加锁
        let _resolve;
        lock_promise = new Promise(resolve => {
            _resolve = resolve;
        });
        try {
            //执行异步任务
            let res = await task();
            return res;
        }finally{
            //释放锁
            lock_promise = undefined;
            _resolve();
        }
    }
}
```



### 测试

```js
serialPack(async () => {return await asyncMethod(1, 2000)}).then(res => {console.log('执行', res)})
serialPack(async () => {return await asyncMethod(2, 2000)}).then(res => {console.log('执行', res)})
serialPack(async () => {return await asyncMethod(3, 2000)}).then(res => {console.log('执行', res)})
serialPack(async () => {return await asyncMethod(4, 2000)}).then(res => {console.log('执行', res)})

function asyncMethod(index, ms){
    return new Promise(resolve => {
        setTimeout(() => resolve(index), ms)
    })
}

//执行结果
/*
竞争锁 ③
执行 1
竞争锁 ②
执行 2
竞争锁
执行 3
执行 4
*/
```

