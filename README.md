#### stage1

> yield 接受 thunk 函数 or Promise 对象

```js
// 自动执行 generator 流程 
function run(gen){
    const it=gen()
    const traverse=function(data){
        const result=it.next(data)
        // 判断 generator 是否执行完毕
        if(result.done)
            return
        // 是Promise对象
        if(isPromsie(result.value)){
            result.value.then(traverse)            
        }else{
            // 是 thunk 对象
            result.value(traverse)
        }
    }
    traverse()
}

// obj.then 为函数，则认为它是一个 Promise 对象
function isPromsie(obj){
    return 'function'===typeof obj.then
} 
```

test

```js
function* gen(){
    const a=yield Promise.resolve(1)
    console.log(a)
    const b=yield fetchData('baidu.com')
    console.log(b)
}


// 模拟数据请求
function fetchData(url) {
    return function(cb) {
        setTimeout(function() {
            cb(null, { status: 200, data: url })
        }, 1000)
    }
}

run(gen)
```



#### stage2

> 获取 generator 最终返回结果 and 捕获 generator 函数执行过程中的错误

**思路：**

将 generator 的执行过程封装称一个Promise，将最后的结果resolve，将过程中产生的错误reject。

为此，我们需要保证yield 后跟一个thunk函数时，返回的值仍然是一个Promise对象。

```js
// 自动执行 generator 流程 
function run(gen){
    const it=gen()

    return new Promise((resolve,reject)=>{
        const traverse=function(data){

            let result
            try{
                result=it.next(data)
            }catch(e){
                reject(e)
            }

            // 判断 generator 是否执行完毕
            if(result.done)
                return
            
            const value=toPromise(result.value)

            // 不是 pormsie 类型，抛出异常并 reject
            if(!isPromsie(value)){
                try {
                    ret = it.throw('You may only yield a function, promise ' +
                    'but the following object was passed: "' + String(value) + '"');
                } catch (e) {
                    return reject(e);
                }
            }

            value.then(data=>traverse(data))
        }
        traverse()
    })
}

function toPromise(obj){
    if(isPromsie(obj)) return obj
    if('function'===typeof obj) return thunkToPromise(obj)
    // tj 认为 yield 后面跟一个原始类型的值没有意义，这一步将导致报错
    return obj
}

// obj.then 为函数，则认为它是一个 Promise 对象
function isPromsie(obj){
    return 'function'===typeof obj.then
} 

// 将 thunk 函数封装成 promise 对象
function thunkToPromise(thunkFn){
    return new Promise((resolve,reject)=>{
        thunkFn((err,data)=>{
            if(err)
                reject(err)
            else
                resolve(data)
        })
    })
}
```

test

```js
function* gen(){
    const a=yield Promise.resolve(1)
    console.log(a)
    const b=yield fetchData('baidu.com')
    console.log(b)
    const c=yield 2
    console.loh(c)
}

// 模拟数据请求
function fetchData(url) {
    return function(cb) {
        setTimeout(function() {
            cb(null, { status: 200, data: url })
        }, 1000)
    }
}

run(gen)
```





#### co 源码

大致和我们的思路相同，只是增加了 

* gen 类型的判断
* onRejected 的封装

```js
// 第三版
function run(gen) {

    return new Promise(function(resolve, reject) {
        if (typeof gen == 'function') gen = gen();

        // 如果 gen 不是一个迭代器
        if (!gen || typeof gen.next !== 'function') return resolve(gen)

        onFulfilled();

        function onFulfilled(res) {
            var ret;
            try {
                ret = gen.next(res);
            } catch (e) {
                return reject(e);
            }
            next(ret);
        }

        function onRejected(err) {
            var ret;
            try {
                ret = gen.throw(err);
            } catch (e) {
                return reject(e);
            }
            next(ret);
        }

        function next(ret) {
            if (ret.done) return resolve(ret.value);
            var value = toPromise(ret.value);
            if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
            return onRejected(new TypeError('You may only yield a function, promise ' +
                'but the following object was passed: "' + String(ret.value) + '"'));
        }
    })
}

function isPromise(obj) {
    return 'function' == typeof obj.then;
}

function toPromise(obj) {
    if (isPromise(obj)) return obj;
    if ('function' == typeof obj) return thunkToPromise(obj);
    return obj;
}

function thunkToPromise(fn) {
    return new Promise(function(resolve, reject) {
        fn(function(err, res) {
            if (err) return reject(err);
            resolve(res);
        });
    });
}

module.exports = run;
```

