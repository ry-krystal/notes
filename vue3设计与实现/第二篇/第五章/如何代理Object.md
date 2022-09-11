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
这是因为ownKeys拦截函数与get/set拦截函数不同，在set/get中，我们可以得到具体操作的key,但是在ownKeys中，我们只能拿到目标对象target。这也很符合直觉，因为在读写属性值时，总是能够明确地知道当前正在操作哪一个属性，所以只需要在该属性与副作用函数之间建立联系即可。而 __ownKeys用来获取一个对象的所有属于自己的键值，这个操作明显不与任何具体的键进行绑定，因此我们只能够构造唯一的key作为标识，即ITEATE_KEY。__

既然追踪的是ITEATE_KEY，那么相应地，在触发响应的时候也应该触发它才行：
```javascript
  trigger(target, ITEATE_KEY)
```

但是在什么情况下，对数据的操作需要触发与ITEATE_KEY相关联的副作用函数重新执行呢？ 为了够清楚这问题。假设副作用函数内有一段for...in循环：
```javascript
  const obj = {
    foo: 1
  }
  const p = new Proxy(obj, {/*  */})

  effect(() => {
    // for...in 循环
    for(const key in p) {
      console.log(key) // foo
    }
  })
```
副作用函数执行后，会与ITEATE_KEY之间建立响应联系，接下来我们尝试为对象p添加新的属性bar:
```javascript
  p.bar = 2
```
由于对象p原本只有foo属性，因此for...in循环只会执行一次。现在为它添加了新的属性bar,所以for...in循环就会由执行一次变成执行两次。也就是说，当为对象添加新属性时，会对for...in循环产生影响，所以需要触发与ITEATE_KEY相关联的副作用函数重新执行。但是目前的实现还做不到这点。当我们为对象p添加新属性bar时，并没有触发副作用函数重新执行，这是为什么呢？我们来看下现在的set拦截函数的实现：
```javascript
  const p = new Proxy(obj, {
    // 拦截设置操作
    set(target, key, newVal, receiver) {
      // 设置属性值
      const res = Reflect.set(target, key, newVal, receiver)
      // 把副作用函数从桶里取出并执行
      trigger(target, key)

      return res
    },

    // 省略其他拦截函数
  })
```
当为对象p添加新的bar属性时，会触发set拦截函数执行。此时set拦截函数接收到的key就是字符串'bar', 因此最终调用trigger时也只是触发了与'bar'相关联的副作用函数重新执行。但根据前文的介绍，我们知道 __for...in循环只是在副作用函数与ITEATE_KEY之间建立联系，这和'bar'一点关系都没有，因此我们尝试执行p.bar = 2操作时，并不能正确地触发响应。__

弄清楚了问题在哪里，解决方案随之而来。__当添加属性时，我们将那些与ITEATE_KEY相关联的副作用函数也取出来执行__ 就可以了：
```javascript
  function trigger(target, key) {
    const depsMap = bucket.get(target)
    if (!depsMap) return

    // 取得与key相关联的副作用函数
    const effects = depsMap.get(key)
    // 取得与ITEATE_KEY相关的副作用函数
    const iterateEffects = depsMap.get(ITEATE_KEY)

    const effectsToRun = new Set()
    // 将与key相关联的副作用函数添加到effectsToRun
    effects && effects.forEach(effectFn => {
      if (effectFn !== activeEffect) {
        effectsToRun.add(effectFn)
      }
    })

    // 将与ITEATE_KEY相关联的副作用函数也添加到 effectsToRun
    iterateEffects && iterateEffects.forEach(effectFn => {
      if (effectFn !== activeEffect) {
        effectsToRun.add(effectFn)
      }
    })

    effectsToRun.forEach(effectFn => {
      const { scheduler } = effectFn.options
      if (scheduler) { // 存在调度函数
        scheduler(effectFn) // 将effectFn执行的控制权交给用户
      } else {
        effectFn() // 立刻执行effectFn
      }
    })
  }
```
当trigger函数执行时，除了把那些直接与具体操作的key相关联的副作用函数取出来执行外，还要把那些与ITEATE_KEY相关联的副作用函数取出来执行。

相信细心的你已经发现了，对于添加新属性来说，这么做没什么问题，但如果仅仅修改已有的属性值，而不是添加新属性，那么问题就来了，如下所示：
```javascript
  const obj = { foo: 1 }
  const p = new Proxy(obj, {/*  */})

  effect(() => {
    // for...in 循环
    for (const key in p) {
      console.log(key) // foo
    }
  })
```
当我们修改p.foo的值时：
```javascript
  p.foo = 2
```
与添加新属性不同，修改属性不会对for...in循环产生影响。因为无论怎么修改一个属性值，对于for...in循环来说都只会循环一次。所以在这种情况下，我们不需要触发副作用函数重新执行，否则会造成不必要的性能开销。然而 __无论是添加新属性，还是修改已有的属性值,其基本语义都是\[[set]],我们都是通过set拦截函数来实现拦截的__，如下所示：
```javascript
  const p = new Proxy(obj, {
    // 拦截设置操作
    set(target, key, newVal) {
      // 设置属性值
      const res = Reflect.set(target, key, newVal, receiver)
      // 把副作用函数从桶里取出来并执行
      trigger(target, key)

      return res
    },
    // 省略其他拦截函数
  })
```
所以要想解决上述问题，当设置属性操作发生时，就需要我们在set拦截函数内能够区分操作的类型，到底是添加新属性还是设置已有属性：
```javascript
  const p = new Proxy(obj, {
    // 拦截设置操作
    set(target, key, newVal, receiver) {
      // 如果属性不存在，则说明是在添加新属性，否则是设置已有属性
      const type = Object.prototype.hasOwnProperty.call(target, key) ? 'SET' : 'ADD'

      // 设置属性值
      const res = Reflect.set(target, key, newVal, receiver)

      // 将type作为第三个参数传递给trigger函数
      trigger(target, key, type)

      return res
    },
    // 省略其他函数
  })
```
在trigger函数内就可以通过类型type来区分当前的操作类型，并且只有当前操作类型type为'ADD'时，才会触发与ITEATE_KEY相关联的副作用函数重新执行，这样就避免了不必要的性能损耗：
```javascript
  function trigger(target, key, type) {
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set()
    // 先去与key关联的副作用函数
    effects && effects.forEach(effectFn => {
      if (effectFn !== activeEffect) {
        effectsToRun.add(effectFn)
      }
    })
    console.log(type, key)
    // 只有当操作类型为'ADD'时，才触发与ITEATE_KEY相关联的副作用函数重新执行
    if (type === 'ADD') {
      const iterateEffects = depsMap.get(ITEATE_KEY)
      iterateEffects && iterateEffects.forEach(effectFn => {
        if (effectFn !== activeEffect) {
          effectsToRun.add(effectFn)
        }
      })
    }

    effectsToRun.forEach(effectFn => {
      const { scheduler } = effectFn.options
      if (scheduler) { // 存在调度函数
        scheduler(effectFn) // 将effectFn执行的控制权交给用户
      } else {
        effectFn() // 立刻执行effectFn
      }
    })
  }
```
通常我们会将操作封装为一个枚举值，例如：
```javascript
  const TriggerType = {
    SET: 'SET',
    ADD: 'ADD'
  }
```
这样无论是对后期代码的维护，还是对代码的清晰度，都是非常有帮助的。

关于对象的代理，还剩下最后一项工作需要做，即删除属性操作的代理：
```javascript
  delete p.foo
```
如何代理delete操作呢？
查看规范得知，delete操作符的行为依赖\[[Delete]]内部方法，该内部方法可以使用deleteProperty拦截：
```javascript
  const p = new Proxy(obj, {
    deleteProperty(target, key) {
      // 检查被操作的属性是否是对象自己的属性
      const hadKey = Object.prototype.hasOwnProperty.call(target, key)
      // 使用Reflect.deleteProperty完成属性的删除
      const res = Reflect.deleteProperty(target, key)

      if (res && hadKey) {
        // 只有当被删除的属性是对象自己的属性并且成功删除时，才触发更新
        trigger(target, key, 'DELETE')
      }

      return
    }
  })
```
由于删除操作会使对象的键变少，它会影响for...in循环次数，因此当操作类型为'DELETE'时，我们也应该触发那些与ITEATE_KEY相关联的副作用函数重新执行：
```javascript
  function trigger(target, key, type) {
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set()
    // 先去与key关联的副作用函数
    effects && effects.forEach(effectFn => {
      if (effectFn !== activeEffect) {
        effectsToRun.add(effectFn)
      }
    })
    console.log(type, key)
    // 只有当操作类型为'ADD'或 'DELETE'时，需要触发与ITEATE_KEY相关联的副作用函数重新执行
    if (type === 'ADD' || type === 'DELETE') {
      const iterateEffects = depsMap.get(ITEATE_KEY)
      iterateEffects && iterateEffects.forEach(effectFn => {
        if (effectFn !== activeEffect) {
          effectsToRun.add(effectFn)
        }
      })
    }

    effectsToRun.forEach(effectFn => {
      const { scheduler } = effectFn.options
      if (scheduler) { // 存在调度函数
        scheduler(effectFn) // 将effectFn执行的控制权交给用户
      } else {
        effectFn() // 立刻执行effectFn
      }
    })
  }
```