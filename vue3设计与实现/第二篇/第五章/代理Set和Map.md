<!--
 * @Description: 代理Set和Map
-->
#### 代理Set和Map
Map和Set这两个数据类型的操作方法相似。它们之间最大的不同提现在，Set类型使用add(value)方法添加元素，而Map类型使用set(key, value)方法设置键值对，并且Map类型可以使用get(key)方法读取相应的值。既然两者如此相似，那么是不是意味着我们可以用相同的处理办法来实现对它们的代理呢？没错，接下来，我们就深入探讨如何实现Set和Map类型数据的代理。
1. 如何代理Set和Map
前文讲到，Set和Map类型的数据有特定的属性和方法用来操作自身。这一点与普通对象不同，如下面的代码所示：
```javascript
  // 普通对象的读取和设置操作
  const obj = {
    foo: 1
  }
  obj.foo  // 读取属性
  obj.foo = 2 // 设置属性

  // 用get/set方法操作Map数据
  const map = new Map()
  map.set('key', 1) // 设置数据
  map.get('key') // 读取数据
```
正是因为这些差异的存在，我们不能像代理普通对象那样代理Set和Map类型的数据。但整体思路不变，即当读取操作发生时，应该调用track函数建立响应联系；当设置操作发生时，应该调用trigger函数触发响应，例如：
```javascript
  const proxy = reactive(new Map([['key', 1]]))

  effect(() => {
    console.log(proxy.get('key')) // 读取键为key的值
  })

  proxy.set('key', 2) // 修改key的值，应该触发响应
```
当然，这段代码展示的效果是我们最终要实现的目标。但是在动手实现之前，我们有必要先了解关于使用Proxy代理Set或Map类型数据的注意事项。
先看一段代码，如下：
```javascript
  const s = new Set([1, 2, 3])
  const p = new Proxy(s, {})

  console.log(p.size) // Uncaught TypeError: Method get Set.prototype.size called on incompatible receiver
```
我们大概能猜到，size属性应该是一个访问器属性，所以它作为方法被调用了。Set.prototype.size是一个访问器属性，它的set访问器函数是undefined,由于我们是通过代理对象p来访问size属性的，所以this就是代理对象p。接着，调用抽象方法RequireInternalSlot(s, \[[SetData]])来检查代理对象是否存在内部槽\[[SetData]]。很显然，代理对象不存在\[[SetData]]这个内部槽，于是抛出一个错误。

为了修复这个问题，我们需要修正访问器属性的getter函数执行时的this指向，如下面的代码所示：
```javascript
  const s = new Set([1, 2, 3])
  const p = new Proxy(s, {
    get(target, key, receiver) {
      if (key === 'size') {
        // 如果读取的是size属性
        // 通过指定第三个参数receiver为原始对象target从而修复这个问题
        return Reflect.get(target, key, target)
      }
      // 读取其他属性的默认行为
      return Reflect.get(target, key, receiver)
    }
  })
```
我们创建代理对象时增加了get拦截函数。然后检查读取的属性名称是不是size,如果是，则在调用Reflect.get函数时指定第三个参数为原始Set对象，这样访问器属性size的getter函数执行时，其this指向的就是原始Set对象而非代理对象了。由于原始Set对象上存在\[[SetData]]内部槽，因此程序得以正确运行。
接着，我们再来尝试从Set中删除数据，如下所示：
```javascript
  const s = new Set([1, 2, 3])
  const p = new Proxy(s, {
    get(target, key, receiver) {
      if (key === 'size') {
        return Reflect.get(target, key, target)
      }
      // 读取其他属性的默认行为
      return Reflect.get(target, key, receiver)
    }
  })

  // 调用delete方法删除值为1的元素
  // 会得到错误TypeError: Method Set.prototype.delete called on incompatible receiver [object object]
  p.delete(1)
```
访问p.size与访问p.delete是不同的。这是因为size是属性，是一个访问器属性，而delete是一个方法。当访问p.size时，访问器属性的getter函数会立即执行，此时我们可以通过修改receiver来改变getter函数的this指向。而当访问p.delete时，delete方法并没有执行，真正使其执行的语句是p.delete(1)这句函数调用。因此，无论怎么修改receiver,delete方法执行时的this都会指向代理对象p,而不会指向原始Set对象。想要修复这个问题也不难，只需要把delete方法与原始对象绑定即可，如下所示：
```javascript
  const s = new Set([1, 2, 3])
  const p = new Proxy(s, {
    get(target, key, receiver) {
      if (key === 'size') {
        return Reflect.get(target, key, target)
      }
      // 将方法与原始数据对象target绑定后返回
      return target[key].bind(target)
    }
  })

  // 调用delete方法删除值为1的元素，正确执行
  p.delete(1)
```
我们使用target[key].bind(target)代替Reflect.get(target, key, receiver)。可以看到，我们使用bind函数将用于操作数据的方法与原始数据对象target做了绑定。这样当p.delete(1)语句执行时，delete函数的this总是指向原始数据对象而非代理对象。
最后，为了后续方便以及代码的可扩展性，我们将new Proxy也封装到前文介绍的createReactive函数中：
```javascript
  const reactiveMap = new Map()
  // reactive函数与之前相比没有变化
  function reactive(obj) {
    const proxy = createReactive(obj)

    const existionProxy = reactiveMap.get(obj)
    if (existionProxy) return existionProxy

    reactiveMap.set(obj, proxy)

    return proxy
  }
  // 在createReactive里封装用于代理Set/Map类型数据的逻辑
  function createReactive(obj, isShallow = false, isReadonly = false) {
    return new Proxy(obj, {
      get(target, key, receiver) {
        if (key === 'size') {
          return Reflect.get(target, key, target)
        }
        return target[key].bind(target)
      }
    })
  }
```
这样我们就可以简单地创建代理数据了：
```javascript
  const p = reactive(new Set([1, 2, 3]))
  console.log(p.size) // 3
```
2. 建立响应联系
了解了为Set和Map类型数据创建代理时的注意事项之后，我们就可以着手实现Set类型数据的响应式方案了。其实思路并不复杂，以下面的代码为例：
```javascript
  const p = reactive(new Set([1, 2, 3]))

  effect(() => {
    // 在副作用函数内访问size属性
    console.log(p.size)
  })
  // 添加值为1的元素,应该触发响应
  p.add(1)
```
首先，在副作用函数内访问了p.size属性；接着，调用p.add函数向集合中添加数据。由于这个行为会间接改变集合的size属性值，所以我们期望副作用函数会重新执行。为了实现这个目标，我们需要在访问size属性时调用track函数进行依赖追踪，然后在add方法执行时调用trigger函数触发响应。下面的代码展示了如何进行依赖追踪：
```javascript
  function createReactive(obj, isShallow = false, isReadonly = false) {
    return new Proxy(obj, {
      get(target, key, receiver) {
        if (key === 'size') {
          // 调用track函数建立响应联系
          track(target, ITERATE_KEY)
          return Reflect.get(target, key, target)
        }

        return target[key].bind(target)
      }
    })
  }
```
可以看到，当读取size属性时，只需要调用track函数建立联系即可。这里需要注意的是，响应联系需要建立在ITERATE_KEY与副作用函数之间，这是因为任何新增、删除操作都会影响size属性。接着，我们来看如何触发响应。当调用add方法向集合中添加新元素时，应该怎么触发响应呢？很显然，这需要我们实现一个自定义的add方法才对，如下所示：
```javascript
  // 定义一个对象，将自定义的add方法定义到该对象下
  const mutableInstrumentations = {
    add(key) {/*  */}
  }

  function createReactive(obj, isShallow = false, isReadonly = false) {
    return new Proxy(obj, {
      get(target, key, receiver) {
        // 如果读取的是raw属性，则返回原始数据对象target
        get(target, key, receiver) {
          if (key === 'raw') return target
          if (key === 'size') {
            track(target, ITERATE_KEY) // 建立响应联系
            return Reflect.get(target, key, target)
          }
          // 返回定义在mutableInstrumentations对象下的方法
          return mutableInstrumentations[key]
        }
      }
    })
  }
```
首先，定义一个对象mutableInstrumentations，我们会将所有自定义实现的方法都定义到该对象下，例如mutableInstrumentations.add 方法。然后，在get拦截函数内返回定义在mutableInstrumentations对象中的方法。这样，当通过p.add获取方法时，得到的就是我们自定义的mutableInstrumentations.add方法了。有了自定义实现的方法后，就可以在其中调用trigger函数触发响应了：
```javascript
  // 定义一个对象，将自定义的add方法定义到该对象下
  const mutableInstrumentations = {
    add(key) {
      // this仍然指向的是代理对象，通过raw属性获取原始数据对象
      const target = this.raw
      // 通过原始数据对象执行add方法删除具体的值,
      // 注意，这里不再需要.bind了，因为是直接通过target调用并执行的
      const res = target.add(key)
      // 调用trigger函数触发响应，并制定操作类型为ADD
      trigger(target, key, 'ADD')
      // 返回操作结果
      return res
    }
   }
```
回顾trigger函数：
```javascript
  function trigger(target, key, type, newVal) {
    const depsMap = bucket.get(target)
    if (!depsMap) return
    const effects = depsMap.get(key)

    // 省略无关内容

    // 当操作类型type为ADD时，会取出与ITERATE_KEY相关联的副作用函数并执行
    if(type === 'ADD' || type === 'DELETE') {
      const iterateEffects = depsMap.get(ITERATE_KEY)
      iterateEffects?.forEach(effectFn => {
        if (effectFn !== activeEffect) {
          effectsToRun.add(effectFn)
        }
      })
    }

    effectsToRun.forEach(effectFn => {
      if (effectFn.options.scheduler) {
        effectFn.options.scheduler(effectFn)
      } else {
        effectFn()
      }
    })
  }
```
当操作类型时ADD或DELETE时，会取出与ITERATE_KEY相关联的副作用函数并执行，这样就可以触发访问size属性所收集的副作用函数来执行了。

当然，如果调用add方法添加的元素已经存在于Set集合中，就不需要触发响应了，这样做对性能更加友好了，因此，我们可以对代码做如下优化：
```javascript
  const mutableInstrumentations = {
    add(key) {
      const target = this.raw
      // 先判断值是否已经存在
      const hadKey = target.has(key)
      // 只有在值不存在的情况下，才需要触发响应
      if(!hadKey) {
        const res = target.add(key)
        trigger(target, key, 'ADD')
      }
      return res
    }
  }
```
在此基础上，我们可以按照类似的思路轻松实现delete方法：
```javascript
  const mutableInstrumentations = {
    delete(key) {
      const target = this.raw
      const hadKey = target.has(key)
      const res = target.delete(key)
      // 当要删除的元素确实存在时，才触发响应
      if (hadKey) {
        trigger(target, key, 'DELETE')
      }
      return res
    }
  }
```
与add 方法的区别在于，delete方法只有在要删除的元素确实在集合中存在时，才需要触发响应，这一点恰好于add方法相反。

3. 避免污染原始数据
  我们借助Map类型数据的set和get这两个方法来讲解什么是“避免污染原始数据”及其原因。
  Map数据类型拥有get和set这两个方法，当调用get方法读取数据时，需要调用track函数追踪依赖建立响应联系；当调用set方法设置数据时，需要调用trigger方法触发响应。如下：
  ```javascript
    const p = reactive(new Map[['key', 1]])

    effect(() => {
      console.log(p.get('key'))
    })

    p.set('key', 2)
  ```