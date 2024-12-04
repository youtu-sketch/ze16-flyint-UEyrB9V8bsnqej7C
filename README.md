
# 为什么需要JS沙箱


想象一下🧐


当一个应用（比如应用 A）加载时，可能会对 `window` 对象的属性进行修改或添加。如果不加控制，这些修改会影响到之后加载的其他应用（比如应用 B），就会导致属性读写冲突


**所以！对于各应用的 js文件来说，就需要一个独立的环境来执行，防止 `window` 全局对象发生属性读写冲突，这个独立的执行环境就叫做 JS沙箱**


# 乾坤沙箱


乾坤目前存在三种 JS隔离机制，分别是 `SnapshotSandbox`、`LegacySandbox` 和 `ProxySandbox`


1. **SnapshotSandbox（快照沙箱）**
这是乾坤最早期的沙箱机制，在每次应用激活和失活时遍历 `window` 对象的所有属性，记录并恢复其状态 。
性能很差，浪费内存；优点是可以兼容不支持 `Proxy` 的旧版浏览器
2. **LegacySandbox（单应用代理沙箱）**
在 SnapshotSandbox 基础上进行优化的一种沙箱机制，通过 ES6 的`Proxy` 对 `window` 对象进行更高效的代理。
然而，它仍会读写 `window` 对象，存在全局污染的问题；并且只能支持单个微应用的运行，意味着在一个页面上不能同时运行多个微应用
事实上，LegacySandbox 在未来应该会消失， 逐渐被能够同时支持多个微应用的 ProxySandbox 所取代
3. **ProxySandbox（多应用代理沙箱）**
是乾坤最先进的一种 JS沙箱隔离机制，通过 `Proxy` 对象为每个微应用创建了一个独立的虚拟 `window`。不会操作 `window` 对象，不存在全局污染的问题；而且在同一页面上也支持多个微应用的同时运行
缺点则是不兼容不支持 `proxy` 旧版浏览器


## SnapshotSandbox（快照沙箱）


这是乾坤最早期的沙箱机制，其主要目的是通过记录和恢复 `window` 对象的状态，确保每个微应用在激活和失活时都能够拥有独立的环境


这里我们实现一个简单的快照沙箱机制， 用于隔离不同微应用对全局 `window` 对象的修改，对应乾坤的源代码可以看这里 \- [SnapshotSandbox源代码](https://github.com)


### 工作原理💯


当沙箱激活时， 保存当前 `window` 的状态到 `windowSnapShot`，然后恢复到上次失活前的状态 （从 `modifyPropsMap` 中读取）


当沙箱失活时， 记录所有对 `window` 对象的修改（与 `windowSnapShot` 的差异），并将 `window` 恢复到最初的快照状态



```
class SnapshotSandbox {
  constructor() {
    this.windowSnapShot = {}; // 存储 window 对象的初始快照
    this.modifyPropsMap = {}; // 存储全局哪些属性被修改了
  }
  // 激活
  active() {
    this.windowSnapShot = {};
    // 记录应用 A window 初始状态
    Object.keys(window).forEach(prop => {
      this.windowSnapShot[prop] = window[prop]
    })
    // 恢复到应用 A 上次失活之前的状态
    Object.keys(this.modifyPropsMap).forEach(prop => {
      window[prop] = this.modifyPropsMap[prop]
    })
  }
  // 失活
  inactive() {
    this.modifyPropsMap = {}
    Object.keys(window).forEach(prop => {
      if (window[prop] !== this.windowSnapShot[prop]) {
        this.modifyPropsMap[prop] = window[prop]; // 记录应用A 所做的所有修改
        window[prop] = this.windowSnapShot[prop]; // 将 window 恢复到最初状态
      }
    })
  }
}
let sandbox = new SnapshotSandbox();
sandbox.active();
window.a = 100;
window.b = 200;
sandbox.inactive();
console.log(window.a, window.b)
sandbox.active();
console.log(window.a, window.b)
```

* **性能很差，** 因为需要在每次应用激活和失活时遍历 `window` 对象的所有属性来记录和恢复其状态。在属性较多或频繁切换应用的情况下，性能瓶颈尤为明显。
* **浪费内存，** 保存 `window` 对象的完整状态会占用大量内存
* **优点是可以兼容不支持 `Proxy` 的旧版浏览器**


总的来说，快照沙箱通过拍摄 `window` 对象的快照来实现状态的隔离，虽然简单有效，但存在性能和内存方面的不足，适用于对浏览器兼容性有要求的场景


## LegacySandbox（单应用代理沙箱）


是在 SnapshotSandbox 基础上优化的一种沙箱机制。它利用 ES6 引入的 `Proxy` 特性来提高性能，并实现类似于快照沙箱的功能


这里我们实现一个简单的单应用代理沙箱机制， 对应乾坤的源代码可以看这里 \- [LegacySandbox源代码](https://github.com)


### 工作原理💯


创建一个 `fakeWindow` 对象，并使用 `Proxy` 对其进行代理，在 `set` 拦截器中，`addedPropsMap` 记录新添加的属性，`modifyPropsMap` 记录被修改的属性原始值，`currentPropsMap` 记录所有被添加修改的属性最新值


当沙箱激活时，恢复到上次失活前的状态 （从 `currentPropsMap` 中读取）


当沙箱失活时，还原修改前的属性值并删除新添加的属性，将 `window` 恢复到最初状态（从 `modifyPropsMap`、`addedPropsMap` 中读取）



```
class LegacySandbox {
  constructor() {
    this.modifyPropsMap = new Map() // 存储被修改过的属性原始值
    this.addedPropsMap = new Map() // 存储新添加的属性值
    this.currentPropsMap = new Map() // 存储所有被修改或新添加的属性最新值

    // 创建一个 fakeWindow 对象，并使用 Proxy 对其进行代理
    const fakeWindow = Object.create(null)
    const proxy = new Proxy(fakeWindow, {
      get: (target, key, recevier) => {
        return window[key]
      },
      set: (target, key, value) => {
        if (!window.hasOwnProperty(key)) {
          this.addedPropsMap.set(key, value) // 新添加的属性
        } else if (!this.modifyPropsMap.has(key)) {
          this.modifyPropsMap.set(key, window[key]) // 被修改的属性原始值
        }
        this.currentPropsMap.set(key, value) // 所有的添加修改操作都存储一份最新值
        window[key] = value
      },
    })
    this.proxy = proxy
  }
  // 设置 window 对象的属性
  setWindowProp(key, value) {
    if (value == undefined) {
      delete window[key]
    } else {
      window[key] = value
    }
  }
  // 激活沙箱
  active() {
    // 恢复到应用 A 上次失活之前的状态
    this.currentPropsMap.forEach((value, key) => {
      this.setWindowProp(key, value)
    })
  }
  // 失活沙箱
  inactive() {
    // 被修改的属性重置为原始值
    this.modifyPropsMap.forEach((value, key) => {
      this.setWindowProp(key, value)
    })
    // 移除新添加的属性
    this.addedPropsMap.forEach((value, key) => {
      this.setWindowProp(key, undefined)
    })
  }
}
let sandbox = new LegacySandbox()
sandbox.proxy.a = 100
console.log(window.a, sandbox.proxy.a)
sandbox.inactive()
console.log(window.a, sandbox.proxy.a)
sandbox.active()
console.log(window.a, sandbox.proxy.a)
```

* **性能优化**：通过 `Proxy` 对 `window` 对象进行代理 ，避免了遍历 `window` 的性能开销
* **全局污染**：尽管使用了 `Proxy`，LegacySandbox 依然在全局 `window` 上进行操作，还是会污染全局的 `window`
* **单应用支持**：和 SnapshotSandbox 一样，LegacySandbox 仅支持单个微应用的运行，无法在同一页面同时隔离多个微应用


事实上，LegacySandbox 在未来应该会消失， 逐渐被能够同时支持多个微应用的 ProxySandbox 所取代


## ProxySandbox（多应用代理沙箱）


ProxySandbox 也是一种基于 `Proxy` 对象实现的沙箱机制，它能够支持在同一页面上运行多个微应用，同时保证每个应用对全局环境（`window` 对象）的修改是隔离的


这里我们实现一个简单的多应用代理沙箱机制， 对应乾坤的源代码可以看这里 \- [ProxySandbox源代码](https://github.com)


### 工作原理💯


创建一个 `fakeWindow` 对象，并使用 `Proxy` 对其进行代理


当沙箱激活时，所有的修改都被限制在 `fakeWindow` 内，而不影响真实的 `window` 对象


当沙箱失活时，将 `this.running` 设置为 `false`， 沙箱不再拦截修改操作



```
class ProxySandbox {
  constructor() {
    this.running = false
    // 使用 Proxy 对 fakeWindow 进行代理
    const fakeWindow = Object.create(null)
    this.proxy = new Proxy(fakeWindow, {
      get: (target, key) => {
        return key in target ? target[key] : window[key]
      },
      set: (target, key, value) => {
        if (this.running) {
          target[key] = value // 将修改操作应用到 fakeWindow 上，而不是真实的 window 对象
        }
        return true
      },
    })
  }
  active() {
    if (!this.running) this.running = true
  }
  inactive() {
    this.running = false
  }
}
let sandbox1 = new ProxySandbox()
let sandbox2 = new ProxySandbox()
sandbox1.active()
sandbox2.active()
sandbox1.proxy.a = 100
sandbox2.proxy.a = 100 
console.log(sandbox1.proxy.a, sandbox2.proxy.a)
sandbox1.inactive()
sandbox2.inactive()
sandbox1.proxy.a = 200
sandbox2.proxy.a = 200
console.log(sandbox1.proxy.a, window.a)
console.log(sandbox2.proxy.a, window.a)
```

* **多应用支持**：每个应用拥有自己的虚拟 `window`，这使得多个微应用可以在同一页面中独立运行而不会相互影响
* **高效性能**：通过 `Proxy` 对 `window` 对象进行代理，能够在不修改原始 `window` 的情况下实现沙箱隔离，性能更高。
* **依赖现代浏览器**：依赖于 ES6 的 `Proxy`，不支持较旧的浏览器


ProxySandbox 通过 `Proxy` 为每个微应用创建了一个独立的虚拟 `window`，有效地隔离了微应用之间的全局状态。它是现代微前端架构中实现多应用支持和环境隔离的关键技术之一。通过这种方式，开发者可以在同一页面中并行运行多个微应用，而不用担心全局变量污染或应用间的干扰。


# 参考文档


[GitHub \- zuopf769/qiankun\-js\-sandbox: 乾坤的JS沙箱隔离机制原理剖析](https://github.com)


[GitHub \- careyke/frontend\_knowledge\_structure: qiankun中JS沙箱的实现](https://github.com)


[微前端01 : 乾坤的Js隔离机制原理剖析（快照沙箱、两种代理沙箱）](https://github.com)


  * [为什么需要JS沙箱](#%E4%B8%BA%E4%BB%80%E4%B9%88%E9%9C%80%E8%A6%81js%E6%B2%99%E7%AE%B1)
* [乾坤沙箱](#%E4%B9%BE%E5%9D%A4%E6%B2%99%E7%AE%B1)
* [SnapshotSandbox（快照沙箱）](#snapshotsandbox%E5%BF%AB%E7%85%A7%E6%B2%99%E7%AE%B1)
* [工作原理💯](#%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86):[CMESPEED\-楚门加速器](https://cmnspeed.com)
* [LegacySandbox（单应用代理沙箱）](#legacysandbox%E5%8D%95%E5%BA%94%E7%94%A8%E4%BB%A3%E7%90%86%E6%B2%99%E7%AE%B1)
* [工作原理💯](#%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-1)
* [ProxySandbox（多应用代理沙箱）](#proxysandbox%E5%A4%9A%E5%BA%94%E7%94%A8%E4%BB%A3%E7%90%86%E6%B2%99%E7%AE%B1)
* [工作原理💯](#%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-2)
* [参考文档](#%E5%8F%82%E8%80%83%E6%96%87%E6%A1%A3)

   \_\_EOF\_\_

   ![](https://github.com/burc)柏成  - **本文链接：** [https://github.com/burc/p/18568342](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
