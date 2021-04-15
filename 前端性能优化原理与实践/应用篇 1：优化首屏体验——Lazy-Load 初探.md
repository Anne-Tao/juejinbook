# 优化首屏体验——Lazy-Load 初探

首先要告诉大家的是，截止到上个章节，我们需要大家绞尽脑汁去理解的“硬核”操作基本告一段落了。从本节开始，我们会一起去实现一些必知必会、同时难度不大的常用优化手段。

这部分内容不难，但**很关键**。尤其是近期有校招或跳槽需求的同学，还请务必对这部分内容多加留心，说不定下一次的面试题里就有它们的身影。

## Lazy-Load 初相见

Lazy-Load，翻译过来是“懒加载”。它是针对图片加载时机的优化：在一些图片量比较大的网站（比如电商网站首页，或者团购网站、小游戏首页等），如果我们尝试在用户打开页面的时候，就把所有的图片资源加载完毕，那么很可能会造成白屏、卡顿等现象，因为图片真的太多了，一口气处理这么多任务，浏览器做不到啊！

但我们再想，用户真的需要这么多图片吗？不对，用户点开页面的瞬间，呈现给他的只有屏幕的一部分（我们称之为首屏）。只要我们可以在页面打开的时候把首屏的图片资源加载出来，用户就会认为页面是没问题的。至于下面的图片，我们完全可以等用户下拉的瞬间再即时去请求、即时呈现给他。这样一来，性能的压力小了，用户的体验却没有变差——这个延迟加载的过程，就是 Lazy-Load。

现在我们打开掘金首页：

![](https://user-gold-cdn.xitu.io/2018/10/5/16642f37c7f20143?w=1424&h=1092&f=png&s=251118)

大家留意一栏文章右侧可能会出现的图片，这里咱们给个特写：

![](https://user-gold-cdn.xitu.io/2018/10/5/16642f57149017c9?w=202&h=662&f=png&s=59534)

大家现在以尽可能快的速度，疯狂向下拉动页面。发现什么？是不是发现我们图示的这个图片的位置，会出现闪动——有时候我们明明已经拉到目标位置了，文字也呈现完毕了，图片却慢半拍才显示出来。这是因为，掘金首页也采用了懒加载策略。当我们的页面并未滚动至包含图片的 div 元素所在的位置时，它的样式是这样的：

![](https://user-gold-cdn.xitu.io/2018/10/5/16642f9403b5f320?w=1312&h=142&f=png&s=62803)

我们把代码提出来看一下：

```
<div data-v-b2db8566="" 
    data-v-009ea7bb="" 
    data-v-6b46a625=""   
    data-src="https://user-gold-cdn.xitu.io/2018/9/27/16619f449ee24252?imageView2/1/w/120/h/120/q/85/format/webp/interlace/1"    
    class="lazy thumb thumb"    
    style="background-image: none; background-size: cover;">  
</div>

```

我们注意到 style 内联样式中，背景图片设置为了 none。也就是说这个 div 是没有内容的，它只起到一个**占位**的作用。

这个“占位”的概念，在这个例子里或许体现得不够直观。最直观的应该是淘宝首页的 HTML Preview 效果：

![](https://user-gold-cdn.xitu.io/2018/10/5/1664307daf51a16e?w=2290&h=972&f=png&s=125909)

我们看到，这个还没来得及被图片填充完全的网页，是用大大小小的空 div 元素来占位的。掘金首页也是如此。

一旦我们通过滚动使得这个 div 出现在了可见范围内，那么 div 元素的内容就会发生变化，呈现如下的内容：

![](https://user-gold-cdn.xitu.io/2018/10/5/1664300e3f14b7c9?w=1198&h=252&f=png&s=109226)

我们给 style 一个特写：

```
style="background-image: url(&quot;https://user-gold-cdn.xitu.io/2018/9/27/16619f449ee24252?imageView2/1/w/120/h/120/q/85/format/webp/interlace/1&quot;); background-size: cover;"

```

可以看出，style 内联样式中的背景图片属性从 none 变成了一个在线图片的 URL。也就是说，出现在可视区域的瞬间，div 元素的内容被即时地修改掉了——它被写入了有效的图片 URL，于是图片才得以呈现。这就是懒加载的实现思路。

## 一起写一个 Lazy-Load 吧！

基于上面的实现思路，我们完全可以手动实现一个属于自己的 Lazy-Load。

（**此处敲黑板划重点，Lazy-Load 的思路及实现方式为大厂面试常考题，还望诸位同学引起重视**）

首先新建一个空项目，目录结构如下：

![](https://user-gold-cdn.xitu.io/2018/10/5/1664322fcf90be3e?w=466&h=116&f=png&s=14252)

大家可以往 images 文件夹里塞入各种各样自己喜欢的图片。

我们在 index.html 中，为这些图片预置 img 标签：

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Lazy-Load</title>
  <style>
    .img {
      width: 200px;
      height:200px;
      background-color: gray;
    }
    .pic {
      // 必要的img样式
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="img">
      // 注意我们并没有为它引入真实的src
      <img class="pic" alt="加载中" data-src="./images/1.png">
    </div>
    <div class="img">
      <img class="pic" alt="加载中" data-src="./images/2.png">
    </div>
    <div class="img">
      <img class="pic" alt="加载中" data-src="./images/3.png">
    </div>
    <div class="img">
      <img class="pic" alt="加载中" data-src="./images/4.png">
    </div>
    <div class="img">
      <img class="pic" alt="加载中" data-src="./images/5.png">
    </div>
     <div class="img">
      <img class="pic" alt="加载中" data-src="./images/6.png">
    </div>
     <div class="img">
      <img class="pic" alt="加载中" data-src="./images/7.png">
    </div>
     <div class="img">
      <img class="pic" alt="加载中" data-src="./images/8.png">
    </div>
     <div class="img">
      <img class="pic" alt="加载中" data-src="./images/9.png">
    </div>
     <div class="img">
      <img class="pic" alt="加载中" data-src="./images/10.png">
    </div>
  </div>
</body>
</html>

```

在懒加载的实现中，有两个关键的数值：一个是**当前可视区域的高度**，另一个是**元素距离可视区域顶部的高度**。

**当前可视区域的高度**， 在和现代浏览器及 IE9 以上的浏览器中，可以用 window.innerHeight 属性获取。在低版本 IE 的标准模式中，可以用 document.documentElement.clientHeight 获取，这里我们兼容两种情况：

```
const viewHeight = window.innerHeight || document.documentElement.clientHeight 

```

而**元素距离可视区域顶部的高度**，我们这里选用 getBoundingClientRect() 方法来获取返回元素的大小及其相对于视口的位置。对此 MDN 给出了非常清晰的解释：

> 该方法的返回值是一个 DOMRect 对象，这个对象是由该元素的 getClientRects() 方法返回的一组矩形的集合, 即：是与该元素相关的 CSS 边框集合 。

> DOMRect 对象包含了一组用于描述边框的只读属性——left、top、right 和 bottom，单位为像素。除了 width 和 height 外的属性都是相对于视口的左上角位置而言的。

其中需要引起我们注意的就是 left、top、right 和 bottom，它们对应到元素上是这样的：

![](https://user-gold-cdn.xitu.io/2018/10/5/1664360c6ceda200?w=500&h=500&f=png&s=25422)

可以看出，top 属性代表了元素距离可视区域顶部的高度，正好可以为我们所用！

Lazy-Load 方法开工啦！

```
<script>
    // 获取所有的图片标签
    const imgs = document.getElementsByTagName('img')
    // 获取可视区域的高度
    const viewHeight = window.innerHeight || document.documentElement.clientHeight
    // num用于统计当前显示到了哪一张图片，避免每次都从第一张图片开始检查是否露出
    let num = 0
    function lazyload(){
        for(let i=num; i<imgs.length; i++) {
            // 用可视区域高度减去元素顶部距离可视区域顶部的高度
            let distance = viewHeight - imgs[i].getBoundingClientRect().top
            // 如果可视区域高度大于等于元素顶部距离可视区域顶部的高度，说明元素露出
            if(distance >= 0 ){
                // 给元素写入真实的src，展示图片
                imgs[i].src = imgs[i].getAttribute('data-src')
                // 前i张图片已经加载完毕，下次从第i+1张开始检查是否露出
                num = i + 1
            }
        }
    }
    // 监听Scroll事件
    window.addEventListener('scroll', lazyload, false);
</script>

```

## 小结

本节我们实现出了一个最基本的懒加载功能。但是大家要注意一点：这个 scroll 事件，是一个**危险**的事件——它太容易被触发了。试想，用户在访问网页的时候，是不是可以无限次地去触发滚动？尤其是一个页面死活加载不出来的时候，疯狂调戏鼠标滚轮（或者浏览器滚动条）的用户可不在少数啊！

再回头看看我们上面写的代码。按照我们的逻辑，用户的每一次滚动都将触发我们的监听函数。函数执行是吃性能的，频繁地响应某个事件将造成大量不必要的页面计算。因此，我们需要针对那些有可能被频繁触发的事件作进一步地优化。这里就引出了我们下一节的两位主角——throttle 与 debounce。