<!--
 * @Description: 理解Proxy与Reflect
 * @version: 1.0
 * @Author: renyong
 * @Date: 2022-07-10 23:36:10
 * @LastEditors: Please set LastEditors
 * @LastEditTime: 2022-09-04 14:36:09
-->
#### 5.1 理解Proxy与Reflect
既然Vue3.js的响应式数据是基于Proxy实现的，那么我们就有必要了解Proxy以及与之相关联的Reflect。

什么是Proxy呢？
回答：简单地说，使用Proxy可以创建一个代理对象。__它能够实现对其他对象的代理，这里的关键词是其他对象__，也就是说，Proxy只能代理对象，无法代理非对象，例如字符串、布尔值等。

那么代理指的是什么呢？
回答：__所谓代理，指的就是对一个对象基本语义的代理。它允许我们拦截并重新定义对一个对象的基本操作__。

什么是基本语义？
给出一个对象obj，可以对它进行一些操作，例如读取属性值、设置属性值：
```javascript
  obj.foo // 读取属性foo的值
  obj.foo++ // 读取和设置属性foo的值.
```
类似这种读取、设置属性值的操作。就属于基本语义的操作，即基本操作。既然是基本操作，那么它就可以使用Proxy拦截：
```javascript
  const p = new Proxy(obj, {
    // 拦截读取属性操作
    get() {/*  */},
    // 拦截设置属性操作
    set() {/*  */}
  })
```
在JavaScript的世界里面，万物皆对象。例如一个函数也是一个对象，所以调用函数也是对一个对象的基本操作：
```javascript
  const fn = (name) => {
    console.log('我是：', name)
  }

  // 调用函数是对对象的基本操作
  fn()
```
因此，我们可以用Proxy来拦截函数的调用操作，这里我们使用apply拦截函数的调用：
```javascript
  const p2 = new Proxy(fn, {
    // 使用apply拦截函数调用
    apply(target, thisArg, argArray) {
      target.call(thisArg, ...argArray)
    }
  })

  p2('hcy') // 输出： '我是：hcy'
```
上面两个例子说明了什么是基本操作。Proxy只能够拦截对一个对象的基本操作。那么，什么是非基本操作呢？__其实调用对象下的方法就是典型的非基本操作__，我们叫它 __复合操作__：
```javascript
  obj.fn()
```
实际上，调用一个对象下的方法，是由两个基本语义组成的。__第一个基本语义是get__，即先通过get操作得到obj.fn属性。__第二个基本语义是函数调用__，即通过get得到obj.fn的值后再调用它，也就是我们上面说到的apply。

理解了Proxy，我们再来讨论Reflect。Reflect是一个全局对象，其下有许多方法，例如：
```javascript
  Reflect.get()
  Reflect.set()
  Reflect.apply()
  // ...
```
可能你已经注意到了，Reflect下的方法与Proxy的拦截器方法名字相同，其实这不是偶然。任何在Proxy的拦截器中能够找到的方法，都能够在Reflect中找到同名函数，那么这些函数的作用是什么呢？其实它们的作用一点儿都不神秘。拿Reflect.get函数来说，它的功能就是提供了访问一个对象属性的默认行为，例如下面两个操作是等价的：
```javascript
  const obj = { foo: 1 }
  
  // 直接读取
  console.log(obj.foo) // 1
  // 使用Reflect.get读取
  console.log(Reflect.get(obj, 'foo')) // 1
```
有人会问：既然操作等价，那么它存在的意义是什么呢？实际上 __Reflect.get函数还能接收第三个参数__，即指定接收这receiver，你可以把它理解为函数调用过程中的this，例如：
```javascript
  const obj = {
    get foo() {
      return this.foo
    }
  }
  console.log(Reflect.get(obj, 'foo', { foo: 2 })) // 输出的是2，不是1
```
回顾之前一章实现响应式数据的代码：
```javascript
  const obj = { foo: 1 }
  const p = new Proxy(obj, {
    get(target, key) {
      track(target, key)
      // 注意， 这里我们没有使用Reflect.get完成读取
      return target[key]
    },
    set(target, key, newVal) {
      // 这里同样没有使用Reflect.set完成设置
      target[key] = newVal
      trigger(target, key)
    }
  })
```
那么这段代码有什么问题吗？我们借助effect让问题暴露出来。首先，我们修改一下obj对象，为它添加bar属性：
```javascript
  const obj = {
    foo: 1,
    get bar() {
      return this.foo
    }
  }
```
可以看到，bar属性是一个访问器属性，它返回了this.foo属性的值。接着，我们在effect副作用函数中通过代理对象p访问bar属性：
```javascript
  effect(() => {
    console.log(p.bar) // 1
  })
```
我们分析一下这个过程发生了什么。当effect注册副作用函数执行时，会读取p.bar属性，它会发现p.bar是一个访问器属性，因此执行getter函数。由于在getter函数中通过this.foo读取foo属性值，因此我们认为副作用函数与属性foo之间也会建立联系。当我们修改p.foo的值时应该能够触发响应，使得副作用函数重新执行才对。然而并非如此，我们尝试修改p.foo的值时：
```javascript
  p.foo++
```
副作用函数并没有重新执行，问题出在哪里呢？
实际上，问题就出在bar属性的访问器函数getter里：
```javascript
  const obj = {
    foo: 1,
    get bar() {
      // 这里的this指向的是谁？
      return this.foo
    }
  }
```
当我们使用this.foo读取foo属性值时，这里的this指向的是谁呢？我们回顾一下整个流程。首先，我们通过代理对象p访问p.bar，这会触发代理对象的get拦截函数执行：
```javascript
  const p = new Proxy(obj, {
    get(target, key) {
      track(target, key)
      // 注意，这里我们没有使用Reflect.get完成读取
      return target[key]
    },
    // ...
  })
```
在get拦截函数内，通过target[key]返回属性值。其中target是原始对象obj，而key就是字符串 'bar'，所以target[key]相当于obj.bar。因此，当我们使用p.bar访问bar属性时，它的getter函数内的this指向的其实是原始对象obj，这说明我们最终访问的其实是obj.foo。

很显然，__在副作用函数内通过原始对象访问它的某个属性是不会建立响应联系的__，这等价于：
```javascript
  effect(() => {
    // obj是原始数据，不是代理对象，这样的访问不能够建立响应联系
    obj.foo
  })
```
因为这样做不会建立响应联系，所以出现了无法触发响应的问题。那么这个问题应该如何解决呢？这时Reflect.get函数就派上用场了。先给出解决问题的代码：
```javascript
  const p = new Proxy(obj, {
    // 拦截读取操作，接收第三个参数 receiver
    get(target, key, receiver) {
      track(target, key)
      // 使用Reflect.get返回读取到的属性值
      return Reflect.get(target, key, receiver)
    }
  })
```
上面代码所示，代理对象的get拦截函数接收第三个参数receiver，它代表谁在读取属性，例如：
```javascript
  p.bar // 代理对象p在读取bar属性
```
当我们使用代理对象p访问bar属性时，那么receiver就是p，你可以把它简单地理解为函数调用中的this。接着关键的一步发生了，我们使用Reflect.get(target, key, receiver) 代替之前的target[key]，这里的关键点就是第三个参数receiver。我们已经知道它就是代理对象p，所以访问器属性bar的getter函数内的this指向代理对象p:
```javascript
  const obj = {
    foo: 1,
    get bar() {
      // 现在这里的this为代理对象p
      return this.foo
    }
  }
```
可以看到，this由原始对象obj变成了代理对象p。很显然，这会在副作用函数与响应式数据之前建立响应联系，从而达到依赖收集的效果。如果此时再对p.foo进行自增操作，会发现已经能够触发副作用函数重新执行了。