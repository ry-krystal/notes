<!--
 * @Description: 
-->
#### 如何代理Object
我们将着手实现响应式数据。前面我们使用get拦截函数去拦截对属性的读取操作。但在响应系统中，“读取”是一个很宽泛的概念，例如使用in操作符检查对象上是否具有给定的key也属于“读取”操作，如下所示：
```javascript
  const obj = { foo: 1 }
  effect(() => {
    'foo' in obj
  })
```
响应式系统应该拦截一切读取操作，以便当数据变化时能够正确地触发响应。下面列出了对一个普通对象的所有可能的读取操作。
  - 访问属性：obj.foo
  - 判断对象或原型上是否存在给定的key: key in obj
  - 使用for...in循环遍历对象：for(const key in obj) {}

```javascript
  const obj = { foo: 1 }

  const p = new Proxy(obj, {
    get(target, key, receiver) {
      // 建立联系
      track(target, key)
      // 返回属性值
      return Reflect.get(target, key, receiver)
    }
  })
```
对于in 操作符，应该如何拦截呢？我们尝试找与in操作符对应的拦截函数，__in操作符的运算结果是通过一个叫作HasProperty的抽象方法得到的。关于HasProperty抽象方法，它的返回值是通过调用对象的内部方法\[[HasProperty]]得到的。而\[[HasProperty]]内部方法通过查表找到对应的拦截函数名叫has__,因此我们可以通过has拦截函数实现对in操作符的代理：
```javascript
  const obj = {
    foo: 1
  }
  const p = new Proxy(obj, {
    has(target, key) {
      track (target, key)
      return Reflect.has(target, key)
    }
  })
```
这样，当我们在副作用函数中通过in操作符操作响应式数据时，就能够建立依赖关系：
```javascript
  effect(() => {
    'foo' in p
  })
```

再看如何拦截for...in循环，查看规范，其中关键点在于EnumerateObjectProperties(obj)。这里的EnumerateObjectProperties是一个抽象方法，该方法返回一个迭代器对象，规范给出了满足该抽象方法的示例实现:
```javascript
  function* EnumerateObjectProperties(obj) {
    const visited = new Set()
    // 判断自身的key
    for(const key of Reflect.ownKeys(obj)) {
      if (typeof key === 'symbol') continue
      // 获取该key的属性标识符
      const desc = Reflect.getOwnPropertyDescriptor(obj, key)
      if (desc) {
        // 存在则添加到visited中
        visited.add(key)
        // 该属性是可枚举的
        if(desc.enumerable) yield key
      }
    }

    // 获取obj的原型
    const proto = Reflect.getPrototypeOf(obj)
    if (proto === null) return
    for (const protoKey of EnumerateObjectProperties(proto)) {
        // 如果原型的protoKey不在visited中
      if (!visited.has(protoKey)) yield protoKey
    }
  }
```
可以看到，该方法是一个generator函数，接收一个参数obj。实际上，obj就是被for...in循环遍历的对象，其关键点在于使用Reflect.ownKeys(obj)来获取只属于对象自身拥有的键。因此，我们可以使用ownKeys拦截函数来拦截Reflect.ownKeys操作：
```javascript
  const obj = { foo: 1 }
  const ITEATE_KEY = Symbol()

  const p = new Proxy(obj, {
    ownKeys(target) {
      // 将副作用函数与ITEATE_KEY关联
      track(target, ITEATE_KEY)
      return Reflect.ownKeys(target)
    }
  })
```
上面的代码，拦截ownKeys操作即可拦截for...in循环。我们在使用track函数进行追踪的时候，将ITEATE_KEY作为追踪key,为什么这么做呢？
这是因为ownKeys拦截函数与get/set拦截函数不同，在set/get中，我们可以得到具体操作的key,但是在ownKeys中，我们只能拿到目标对象target。这也很符合直觉，因为在读写属性值时，总是能够明确地知道当前正在操作哪一个属性，所以只需要在该属性与副作用函数之间建立联系即可。而 __ownKeys用来获取一个对象的所有属于自己的键值，这个操作明显不与任何具体的键进行绑定，因此我们只能够构造唯一的key作为标识，即ITEATE_KEY__