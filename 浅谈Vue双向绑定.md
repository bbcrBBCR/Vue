## 浅谈Vue的双向绑定
### 实现原理
  <img src="https://images2015.cnblogs.com/blog/938664/201705/938664-20170522225458132-1434604303.png" width="70%" align=center>
  
Vue的双向绑定是使用数据劫持+发布者-订阅者模式进行的。分为三部分：
* Observer（数据监听器）:进行数据对象的属性监听和更新。
* Compile（模板解析器）:将template编译成render函数的字符串形式。
* Watcher（订阅者）:Compile与Observer之间的桥梁，这意味着对于Compile与Observer是好像不可见的。Watcher的作用在于更新视图以及订阅数据。
### Observer
Observer中的核心方法就是Object.defineProperty()方法，改写Vue对象data中的属性。每一个Observer都创建的一个可以容纳订阅者的消息订阅器Dep，Dep主要负责搜集订阅者，以及在消息变化的时候执行订阅者的更新函数。Dep类的构造函数如下：
```
  var Dep = function Dep () {
    this.id = uid$2++ //Dep的唯一标号
    this.subs = [] //储存订阅者
  };
  
  Dep.prototype.addSub = function addSub (sub) {
    this.subs.push(sub)
  }; //添加订阅者到自己的subs
  
  Dep.prototype.removeSub = function removeSub (sub) {
    remove$1(this.subs, sub)
  }; //从自己的subs中移除订阅者
  
  Dep.prototype.depend = function depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }; //将自己添加到Dep.target指向的Dep.subs中，在Observer的getter中被调用
  
  Dep.prototype.notify = function notify () { 
    // stablize the subscriber list first
    var subs = this.subs.slice()
    for (var i = 0, l = subs.length; i < l; i++) {
      subs[i].update() //调用所有订阅了该数据的watcher的updata方法
    }
  }; //数据更改时调用，当Observer中的数据setter时，调用该函数，更新watcher，然后进一步更新视图
```
从Vue源代码中，可以知道，Dep（消息订阅器或依赖收集器）有3个属性，在初始化时，id是唯一的一个标记、subs为数组、target为null。Dep类是Observer与watcher的桥梁，也就是说，Observer与watcher之间是互相不知道的。
### Watcher
Watcher是将Compile与Observer连接在一起的纽带。Watcher的来源有：代码中的computed属性、watch函数以及模板编译过程中的指令和数据绑定。
```
var Watcher = function Watcher (
    vm, //当前的vue实例
    expOrFn, //传入的getter函数
    cb, //diff算法，一般来自原型链，所以一般传入空函数
    options 
  ) {...}
 Watcher.prototype.get = function get () {
    pushTarget(this) //将watcher压栈
    var value = this.getter.call(this.vm, this.vm) //使用传入的getter方法求值以及依赖收集（依赖收集具体看下回）
    if (this.deep) {
      traverse(value)
    }
    popTarget() //将watcher出栈
    this.cleanupDeps()
    return value
  };
    Watcher.prototype.update = function update () {
    if (this.lazy) {
      this.dirty = true 
    } else if (this.sync) {
      this.run() //同步时使用run进行更新
    } else {
      queueWatcher(this) //异步更新队列，多数情况下使用异步方法（开启一个队列，并缓冲在一个事件循环中发生的所有数据更改。如果同一个Watcher被多次触发，只会别推入一次。避免不必要的计算和DOM操作。在下一个事件喜欢中，执行。）
    }
  };
```
### Compile
Vue 1+使用的是文档片段，Vue 2+使用的是VDOM。Compile的作用是将template编译成render函数的字符串形式，用于创建VNode以及之后的更改DOM操作。主要步骤有：
* parse（解析器）:将template转换成抽象语法树（AST）。
* optimizer（优化器）:对AST进行静态节点的标记，优化渲染过程。
* generateCode（代码生成器）:根据AST生成一个render函数字符串。
#### AST
Vue中的ASTNode分为三类。
* ASTElement：元素
```
var element = {
          type: 1,
          tag: tag,
          attrsList: attrs,
          attrsMap: makeAttrsMap(attrs),
          parent: currentParent,
          children: []
          //...
        }
```
* ASTExpression
```
            //...
              type: 2,
              expression: expression,
              text: text
            //...
```
* ASTText
```
            //...
              type: 3,
              text: text
            //...
```
#### parse
在parse中，核心是parseHTML函数。parseHTML不断的解析模板，填充root，最后把root(AST)返回。在parseHTML中，使用while循环，不断的用indexOf('<')匹配，然后根据不同的返回值进行不同的处理。
* 等于0：代表注释、doctype、开始标签、结束标签等。
* 大于等于0：表示文本、表达式
* 小于0：表示解析完成，可能剩下一些文本、表达式
#### optimizer
主要作用是标记static结点，在patch时可以跳过它们。重要的过程是两次遍历。
* 第一次遍历：markstatic。一个递归的过程，不断的检查AST上的结点，然后进行标记。（当子节点不是static结点时，父节点也不是static）。
```
  function markStatic (node) {
    node.static = isStatic(node) //对node使用isStatic函数，isStatic函数的作用是判断node是不是静态节点
    if (node.type === 1) {
        //type为1时，表示node是元素，有子节点
      for (var i = 0, l = node.children.length; i < l; i++) {
        var child = node.children[i]
        markStatic(child) //判断子节点
        if (!child.static) {
          node.static = false //当子节点不是static结点时，父节点也不是static
        }
      }
    }
  }
    function isStatic (node) {
    if (node.type === 2) { //表达式结点，显然不是static
      return false
    }
    if (node.type === 3) { //文本结点，是static
      return true 
    }
    //使用了v-pre或<pre>，加上pre=true，是静态节点
    //当node是static时，它不可以有动态绑定、v-if、v-for、v-else、不是componment标签等等
    return !!(node.pre || (
      !node.hasBindings && 
      !node.if && !node.for && 
      !isBuiltInTag(node.tag) && 
      !isPlatformReservedTag(node.tag) &&
      Object.keys(node).every(isStaticKey)
    ))
  }
```
* 第二次遍历：markstaticRoots，标记静态根节点（节点是静态的，并且有子节点，而且子节点不是单个静态文本节点）。
#### generateCode
针对AST属性调用不同的方法生成字符串返回，根据AST结构拼接生成render函数的字符串。
### 总结
学无止境。
### 参考
vue的双向绑定原理及实现[https://www.cnblogs.com/libin-1/p/6893712.html](https://www.cnblogs.com/libin-1/p/6893712.html)

Vue源码--深入模板渲染[https://blog.csdn.net/qq_36538012/article/details/80171643](https://blog.csdn.net/qq_36538012/article/details/80171643)

Vue 源码解析：深入响应式原理
[https://github.com/DDFE/DDFE-blog/issues/7](https://github.com/DDFE/DDFE-blog/issues/7)
