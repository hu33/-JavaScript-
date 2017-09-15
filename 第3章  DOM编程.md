##第3章  DOM编程

众所周知，用JS进行DOM操作那是灰常的昂贵，毕竟人家web应用的性能瓶颈之一啊，但也是有不少优化的法子的~~本章大致从三个方向阐述了DOM编程的优化方法。

### 尽量减少访问和修改DOM次数

浏览器中通常是独立实现DOM和JavaScript的，这样可以允许其他技术和语言也能共享使用DOM和渲染函数，但这也意味着JS想去访问DOM，是要花费一定代价的，访问和修改的次数越高，代价就越高，代码运行速度就越慢。

#### 优化点1  使用DOM更新页面内容时，克隆已有元素，而不是创建新元素

在大多数浏览器中，节点克隆都更有效率，虽然效果不是特别明显~~所以对于一些需要多次重复创建的元素，可以先创建第一个，然后重复拷贝操作。也就是用`element.cloneNode()`代替`document.createElement()`。

#### 优化点2  遍历HTML集合时，把集合存在局部变量中，并把length缓存在循环外部，用局部变量代替这些需要多次读取的元素

看个例子就造了：

```javascript
//最慢
function collectionGlobal() {
    var coll = document.getElementByTagName('div'),
    len = coll.length,
    name = '';
    for (var count = 0; count < len; count++) {
        name = document.getElementByTagName('div')[count].nodeName;
        name = document.getElementByTagName('div')[count].nodeType;
        name = document.getElementByTagName('div')[count].tagName;
    }
    return name;
}

//较快
function collectionGlobal() {
    var coll = document.getElementByTagName('div'),
    len = coll.length,
    name = '';
    for (var count = 0; count < len; count++) {
        name = coll[count].nodeName;
        name = coll('div')[count].nodeType;
        name = coll('div')[count].tagName;
    }
    return name;
}

//最快
function collectionGlobal() {
    var coll = document.getElementByTagName('div'),
    len = coll.length,
    name = '',
    ele = null;
    for (var count = 0; count < len; count++) {
        ele = coll[count]; 
        name = ele.nodeName;
        name = ele.nodeType;
        name = ele.tagName;
    }
    return name;
}
```

#### 优化点3  访问元素节点时，推荐使用只返回元素节点的API，因为它们的执行效率比自己在JS代码中实现过滤的效率要高

这些API如下：

- `children` -- `childNodes`
- `childElementCount` -- `childNodes.length`
- `firstElementChild` -- `firstChild`
- `lastElementChild` -- `lastChild`
- `nextElementSibling` -- `nextSibing`
- `previousElementSibling` -- `previousSibing`

#### 优化点4  使用`querySelectorAll()`查询DOM

当咱需要得到特定的DOM元素列表时，需要使用`getElementById()`和`getElementByTagName()`进行组合调用，而且还得手动遍历筛选，好不麻烦且效率低下~~

后来~\~`querySelectorAll()`就横空出世了，只要传入相应的CSS选择器字符串，就能得到咱需要的元素了，而且还是浏览器原生API，比起用JS和DOM操作来遍历查找元素不要快太多啊~\~

而且，`querySelectorAll()`返回的是一个NodeList，是一个快照而不是动态集合，所以还避免了HTML集合会导致的性能问题。

### 尽量避免重绘和重排

浏览器下载完页面中所有的组件后会解析并生成两个内部数据结构--DOM树和Render树，DOM树都知道是表示页面结果，Render树则是表示DOM节点如何显示。Render树中的节点就是一个“盒”，一旦DOM树和Render树构建完成，浏览器就开始绘制元素了。

当DOM的变化影响了元素的几何属性（宽和高），浏览器就需要重新计算元素的几何属性，其他元素的几何属性和位置也会因此受到影响。浏览器会使渲染树中受到影响的元素的部分失效，并重新构建Render树，这就是传说中的重排。重排之后，浏览器会重新绘制受影响的部分，这就是重绘。

易得：重绘和重排都是代价很昂贵的操作，所以要尽量避免避免啊~

#### 优化点5  合并多次对DOM和样式的修改，然后一次处理

- 改变样式：可以用CSSText属性做批量修改
- 批量修改DOM：可以让DOM脱离文档流，修改完再带回文档中，具体有三种方式：
  1. 隐藏元素（`display：none`），修改后再重新显示
  2. 使用文档片段在当前DOM之外构建一个子树，再把它拷贝回文档
  3. 将原始元素拷贝到一个脱离文档流的节点中，修改该副本节点，完成后再替换原始元素

#### 优化点6  缓存布局信息

现在大多数浏览器对重排进行了优化，通过队列化修改并批量修改执行来实现，但是有种操作会强制刷新队列并要求计划任务立即执行！它！就！是！获取元素的布局信息！因为查询布局信息时，浏览器会为了返回最新值而会刷新队列并应用所有变更。

这个时候，局部变量又派上用场了，我们可以通过获取布局信息，把它赋值给局部变量，然后操作局部变量，尽可能地减少布局信息的获取次数。

#### 优化点7  让元素脱离动画流

假设有个元素处在页面顶部，然后它可以展开/折叠，那岂不是对它之后所有的元素位置都会产生影响？那岂不是要进行大规模重排？？那岂不是很消耗性能？？？

咱可以让它脱离文档流啊~先给它设置绝对定位，然后它想咋动就咋动吧，反正就导致一小区域的重绘，等动画结束再恢复它本来的定位。

### 优化点8  使用事件委托

元素绑定事件处理器是有代价的，会占用处理时间，而且浏览器还要跟踪这些处理器，又会占更多的内存。要是页面上一大堆元素都需要绑定一个或多个事件处理器，那还不堵死啊。。。

不过呢，大部分事件是能够冒泡的，咱就可以给外层元素绑定一个处理器，用来处理在其子元素上触发的事件。除了性能上有很大的优化，事件委托其实还有个好处，就是如果一个元素的子元素是动态的，它也可以进行处理。