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

3. 避免污染原数据
  我们借助Map类型数据的set和get两个方法来讲解什么是"避免污染原始数据"及其原因。
  Map数据类型拥有get和set这两个方法，当调用get方法读取数据时，需要调用track函数追踪依赖建立响应联系；当调用set方法设置数据时，需要调用trigger方法触发响应。
  ```javascript
    const p = reactive(new Map([['key', 1]]))
    effect(() => {
      console.log(p.get('key'))
    })

    p.set('key', 2) // 触发响应
  ```
  下面是get方法具体实现：
  ```javascript
    const mutableInstrumentations = {
      get(key) {
        // 获取原始对象
        const target = this.raw
        // 判断读取的key是否存在
        const had = target.has(key)
        // 追踪依赖，建立响应联系
        track(target, key)
        // 如果存在，则返回结果，这里要注意的是，如果得到的结果res仍然是可代理的数据
        if (had) {
          const res = target.get(key)
          return typeof res === 'object' ? reactive(res) : res
        }
      }
    }
  ```
  当set方法被调用时，需要调用trigger方法触发响应。只不过在触发响应的时候，需要区分操作类型是SET还是ADD，如下所示：
  ```javascript
    const mutableInstrumentations = {
      set(key, value) {
        // 获取原始对象
        const target = this.raw
        const had = target.has(key)
        // 获取旧值
        const oldValue = target.get(key)
        // 设置新值
        target.set(key, value)
        // 如果不存在，则说明是ADD类型的操作，意味着新增
        if (!had) {
          trigger(target, key, 'ADD')
        } else if (oldValue !== value || (oldValue === oldValue && value === value)) {
          // 如果存在，并且值变了，则是SET类型操作，意味着修改
          trigger(target, key, 'SET')
        }
      }
    }
  ```
  这段代码的关键点在于，我们需要判断设置的key是否存在，以便区分不同的操作类型。我们知道，__对于SET类型和ADD类型的操作来说，它们最终触发的副作用函数是不同的。因为ADD类型的操作会对数据的size属性产生影响，所以任何依赖size属性的副作用函数都需要在ADD类型的操作发生时重新执行。__

  上面给出的set函数的实现能够正常工作，但它仍然存在问题，即set方法会污染原始数据。这是什么意思呢？来看下面的代码：
  ```javascript
    // 原始Map对象m
    const m = new Map()
    // p1 是 m的代理对象
    const p1 = reactive(m)
    // p2 是另一个代理对象
    const p2 = reactive(new Map())
    // 为p1设置一个键值对，值是代理对象p2
    p1.set('p2', p2) // 相当于给m设置了一个p2,p2也是一个代理的Map

    effect(() => {
      // 注意，这里我们通过原始数据m访问p2
      console.log(m.get('p2').size)
    })
    // 注意，这里我们通过原始数据m为p2设置一个键值对foo --> 1
    m.get('p2').set('foo', 1)
  ```
  在这段代码中，我们首先创建了一个原始Map对象m,p1是对象m的代理对象，接着创建另外一个代理对象p2，并将其作为值设置给p1,即p1.set('p2', p2)。接下来问题就来了，在副作用函数中，我们通过原始数据m来读取数据值，然后又通过原始数据m设置数据值，此时发现副作用函数重新执行了。这其实不是我们所期望的行为，因为原始数据不应该具有响应式数据的能力，否则就意味着用户既可以操作原始数据，又能够操作响应式数据，这样一来代码就乱套了。

  那么导致问题的原因是什么呢？其实很简单，观察我们前面的实现set方法：
  ```javascript
    const mutableInstrumentations = {
      set(key, value) {
        const target = this.raw
        const had = target.has(key)
        const oldValue = target.get(key)
        // 我们把value原封不动地设置到原始数据上
        target.set(key, value)
        if (!had) {
          trigger(target, key, 'ADD')
        } else if (oldValue !== value || (oldValue === oldValue && value === value)) {
          trigger(target, key, 'SET')
        }
      }
    }
  ```
  在set方法内，我们把value原样设置到了原始数据target上。如果value是响应式数据，就意味着设置到原始对象上的也是响应式数据，我们就把
  __响应式数据设置到原始数据上的行为称为数据污染。__

  要解决数据污染也不难，__只需要再调用target.set函数设置值之前对值进行检查即可：只要发现即将设置的值时响应式数据，那么就通过raw属性获取原始数据，再把原始数据设置到target上。__
  ```javascript
    const mutableInstrumentations = {
      set(key, value) {
        const target = this.raw
        const had = target.has(key)

        const oldValue = target.key(key)
        // 获取原始数据，由于value本身可能已经是原始数据，所以此时value.raw不存在，则直接使用value
        const rawValue = value.raw || value
        target.set(key, rawValue)

       if (!had) {
          trigger(target, key, 'ADD')
        } else if (oldValue !== value || (oldValue === oldValue && value === value)) {
          trigger(target, key, 'SET')
        }
      }
    }
  ```
  现在实现已经不会造成数据污染了，不过，细心观察上面的代码，会发现新问题，我们一直用raw属性来访问原始数据是有缺陷的，因为它可能与用户自定义的raw属性冲突，所以在一个严谨的实现中，我们需要使用唯一的标识作为访问原始数据的键，例如使用Symbol类型来替代。
  除了set方法需要避免污染原始数据之外，Set类型的add方法，普通对象的写值操作，还有为数组添加元素的方法等，都需要做类似的处理。

  4. 处理forEach
  集合类型forEach方法类似于数据的forEach方法，我们先看看它是如何工作的：
  ```javascript
    const m = new Map([
      [{ key: 1 }, { value: 1 }]
    ])

    effect(() => {
      m.forEach(function(value, key, m) {
        console.log(value) // { value: 1 }
        console.log(key) // { key: 1 }
      })
    })
  ```
  以Map为例，forEach方法接收一个回调函数作为参数，该回调函数会在Map的每个键值对上被调用。回调函数接收三个参数，分别是值、键以及原始Map对象。如上面的代码所示，我们可以使用forEach方法遍历Map数据的每一组键值对。

  __遍历操作只与键值对的数量有关，因此任何会修改Map对象键值对数量的操作都应该触发副作用函数重新执行，例如delete和add方法等。__ 所以当forEach函数被调用时，我们应该让副作用函数与ITERATE_KEY建立响应联系，如图：
  ```javascript
    const mutableInstrumentations = {
      forEach(callback) {
        // 取得原始数据对象
        const target = this.raw
        // 与ITERATE_KEY建立响应联系
        track(target, ITERATE_KEY)
        // 通过原始数据对象调用forEach方法，并把callback传递过去
        target.forEach(callback)
      }
    }
  ```
  这样我们就实现了forEach操作的追踪，可以使用下面的代码进行测试：
  ```javascript
    const p = reactive(new Map[[
      {key: 1}, {value: 1}
    ]])

    effect(() => {
      p.forEach(function(value, key) {
        console.log(value) // { value: 1 }
        console.log(key) // { key: 1 }
      })
    })

    // 能够触发响应
    p.set({key: 2}, { value: 2})
  ```
  上面给出的forEach函数仍然存在缺陷，我们在自定义实现的forEach方法内，通过原始数据对象调用了原生的forEach方法，即：
  ```javascript
    // 通过原始数据对象调用forEach方法，并把callback传递进去
    target.forEach(callback)
  ```
  这意味着，传递给callback回调函数的参数将是非响应式数据。这导致下面的代码不能按预期工作：
  ```javascript
    const key = { key: 1 }
    const value = new Set([1, 2, 3])
    const p = reactive(new Map[
      [key, value]
    ])

    effect(() => {
      p.forEach(function(value, key) {
        console.log(value.size) // 3
      })
    })

    p.get(key).delete(1)
  ```
  上面的这段代码中，响应式数据p有一个键值对，其中键是普通对象{ key: 1 },值是Set类型的原始数据new Set([1, 2, 3])。接着，我们在副作用函数中使用forEach方法遍历p,并在回调函数中访问value.size。最后，我们尝试删除Set类型数据中值为1的元素，却发现没能触发副作用函数重新执行。导致问题的原因就是上面曾提到的，当通过value.size访问size属性时，这里的value是原始数据对象，即new Set([1, 2, 3])，而非响应式数据对象，因此无法建立响应联系。但这其实不符合直觉，因为reactive本身是深响应，forEach方法的回调函数所接收到的参数也应该是响应式数据才对。为了解决这个问题，我们需要对现有实现做一些修改，如下所示：
  ```javascript
    const mutableInstrumentations = {
      forEach(callback) {
        // wrap函数用来把可代理的值转换为响应式数据
        const wrap = (val) => typeof val === 'object' ? reactive(val) : val
        const target = this.raw
        track(target, ITERATE_KEY)
        // 通过target调用原始forEach方法进行遍历
        target.forEach((v, k) => {
          // 手动调用callback, 用wrap函数包裹value和key后再传给callback,这样就实现了深响应
          callback(wrap(v), wrap(k), this)
        })
      }
    }
  ```
  forEach函数除了接收callback作为参数之外，它还接收第二个参数，该参数可以用来指定callback函数执行时的this值。更加完善的实现如下：
  ```javascript
    const mutableInstrumentations = {
      forEach(callback, thisArg) {
        // wrap函数用来把可代理的值转换为响应式数据
        const wrap = (val) => typeof val === 'object' ? reactive(val) : val
        const target = this.raw
        track(target, ITERATE_KEY)
        // 通过target调用原始forEach方法进行遍历
        target.forEach((v, k) => {
          // 通过.call调用callback,并传递thisArg
          callback.call(thisArg, wrap(v), wrap(k), this)
        })
      }
    }
  ```
  然而，使用for...in来遍历对象与使用forEach遍历集合之间存在本质的不同。具体体现在，当使用for...in循环对象时，它只关心对象的键，而不关心对象的值
  ```javascript
    effect(() => {
      for(const key in obj) {
        console.log(key)
      }
    })
  ```
  只有当新增、删除对象的key时，才需要重新执行副作用函数。所以我们在trigger函数内判断操作类型是否是ADD或DELETE，进而知道是否需要触发那些与ITERATE_KEY相关联的副作用函数重新执行。对于SET类型操作来说，因为它不会改变一个对象的键的数量，所以当SET类型的操作发生时，不需要触发副作用函数重新执行。
    但这个规则不适用于Map类型的forEach遍历，如下：
  ```javascript
    const p  = reactive(new Map([
      ['key', 1]
    ]))

    effect(() => {
      p.forEach(function(value, key) {
        // forEach循环不仅关心集合的键，还关心集合的值
        console.log(value) // 1
      })
    })

    p.set('key', 2) // 即使操作类型是SET，也应该触发响应
  ```
  当使用forEach遍历Map类型的数据时，它机既关心键，又关心值。这意味着，当调用p.set('key', 2)修改值的时候，也应该触发副作用函数重新执行，即使它的操作类型是SET。因此，我们应该修改trigger函数的代码来弥补这个缺陷：
  ```javascript
    function trigger(target, key, type, newVal) {
      console.log('trigger', key)
      const depsMap = bucket.get(target)
      if (!depsMap) return
      const effects = depsMap.get(key)

      const effectsToRun = new Set()
      effects && effects.forEach(effectFn => {
        if (effectFn !== activeEffect) {
          effectsToRun.add(effectFn)
        }
      })

      if (
        type === 'ADD' || 
        type === 'DELETE' ||
        // 如果操作类型是SET，并且目标对戏那个是Map类型的数据，
        // 也应该触发那些与ITERATE_KEY相关联的副作用函数重新执行
        (
          type === 'SET' &&
          Object.prototype.toString.call(target) === '[object Map]'
        )
      ) {
        const iterateEffects = depsMap.get(ITERATE_KEY)
        iterateEffects?.forEach(effectFn => {
          if (effectFn !== activeEffect) {
            effectsToRun.add(effectFn)
          }
        })
      }

      // ...

      effectsToRun.forEach(effectFn => {
        if (effectFn.options.scheduler) {
          ffectFn.options.scheduler(effectFn)
        } else {
          effectFn()
        }
      })
    }
  ```