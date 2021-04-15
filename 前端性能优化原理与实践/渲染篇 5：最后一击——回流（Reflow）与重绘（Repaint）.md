# 最后一击——回流（Reflow）与重绘（Repaint）

开篇我们先对上上节介绍的回流与重绘的基础知识做个复习（跳读的同学请自觉回到上上节补齐 →\_→）。

**回流**：当我们对 DOM 的修改引发了 DOM 几何尺寸的变化（比如修改元素的宽、高或隐藏元素等）时，浏览器需要重新计算元素的几何属性（其他元素的几何属性和位置也会因此受到影响），然后再将计算的结果绘制出来。这个过程就是回流（也叫重排）。

**重绘**：当我们对 DOM 的修改导致了样式的变化、却并未影响其几何属性（比如修改了颜色或背景色）时，浏览器不需重新计算元素的几何属性、直接为该元素绘制新的样式（跳过了上图所示的回流环节）。这个过程叫做重绘。

由此我们可以看出，**重绘不一定导致回流，回流一定会导致重绘**。硬要比较的话，回流比重绘做的事情更多，带来的开销也更大。但这两个说到底都是吃性能的，所以都不是什么善茬。我们在开发中，要从代码层面出发，尽可能把回流和重绘的次数最小化。

## 哪些实际操作会导致回流与重绘

要避免回流与重绘的发生，最直接的做法是避免掉可能会引发回流与重绘的 DOM 操作，就好像拆弹专家在解决一颗炸弹时，最重要的是掐灭它的导火索。

触发重绘的“导火索”比较好识别——只要是不触发回流，但又触发了样式改变的 DOM 操作，都会引起重绘，比如背景色、文字色、可见性(可见性这里特指形如visibility: hidden这样不改变元素位置和存在性的、单纯针对可见性的操作，注意与display:none进行区分)等。为此，我们要着重理解一下那些可能触发回流的操作。

### 回流的“导火索”

*   最“贵”的操作：改变 DOM 元素的几何属性

这个改变几乎可以说是“牵一发动全身”——当一个DOM元素的几何属性发生变化时，所有和它相关的节点（比如父子节点、兄弟节点等）的几何属性都需要进行重新计算，它会带来巨大的计算量。

常见的几何属性有 width、height、padding、margin、left、top、border 等等。此处不再给大家一一列举。有的文章喜欢罗列属性表格，但我相信我今天列出来大家也不会看、看了也记不住（因为太多了）。我自己也不会去记这些——其实确实没必要记，️一个属性是不是几何属性、会不会导致空间布局发生变化，大家写样式的时候完全可以通过代码效果看出来。多说无益，还希望大家可以多写多试，形成自己的“肌肉记忆”。

*   “价格适中”的操作：改变 DOM 树的结构

这里主要指的是节点的增减、移动等操作。浏览器引擎布局的过程，顺序上可以类比于树的前序遍历——它是一个从上到下、从左到右的过程。通常在这个过程中，当前元素不会再影响其前面已经遍历过的元素。

*   最容易被忽略的操作：获取一些特定属性的值

当你要用到像这样的属性：offsetTop、offsetLeft、 offsetWidth、offsetHeight、scrollTop、scrollLeft、scrollWidth、scrollHeight、clientTop、clientLeft、clientWidth、clientHeight 时，你就要注意了！

“像这样”的属性，到底是像什么样？——这些值有一个共性，就是需要通过**即时计算**得到。因此浏览器为了获取这些值，也会进行回流。

除此之外，当我们调用了 getComputedStyle 方法，或者 IE 里的 currentStyle 时，也会触发回流。原理是一样的，都为求一个“即时性”和“准确性”。

## 如何规避回流与重绘

了解了回流与重绘的“导火索”，我们就要尽量规避它们。但很多时候，我们不得不使用它们。当避无可避时，我们就要学会更聪明地使用它们。

### 将“导火索”缓存起来，避免频繁改动

有时我们想要通过多次计算得到一个元素的布局位置，我们可能会这样做：

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
  <style>
    #el {
      width: 100px;
      height: 100px;
      background-color: yellow;
      position: absolute;
    }
  </style>
</head>
<body>
  <div id="el"></div>
  <script>
  // 获取el元素
  const el = document.getElementById('el')
  // 这里循环判定比较简单，实际中或许会拓展出比较复杂的判定需求
  for(let i=0;i<10;i++) {
      el.style.top  = el.offsetTop  + 10 + "px";
      el.style.left = el.offsetLeft + 10 + "px";
  }
  </script>
</body>
</html>

```

这样做，每次循环都需要获取多次“敏感属性”，是比较糟糕的。我们可以将其以 JS 变量的形式缓存起来，待计算完毕再提交给浏览器发出重计算请求：

```
// 缓存offsetLeft与offsetTop的值
const el = document.getElementById('el') 
let offLeft = el.offsetLeft, offTop = el.offsetTop

// 在JS层面进行计算
for(let i=0;i<10;i++) {
  offLeft += 10
  offTop  += 10
}

// 一次性将计算结果应用到DOM上
el.style.left = offLeft + "px"
el.style.top = offTop  + "px"

```

### 避免逐条改变样式，使用类名去合并样式

比如我们可以把这段单纯的代码：

```
const container = document.getElementById('container')
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'

```

优化成一个有 class 加持的样子：

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
  <style>
    .basic_style {
      width: 100px;
      height: 200px;
      border: 10px solid red;
      color: red;
    }
  </style>
</head>
<body>
  <div id="container"></div>
  <script>
  const container = document.getElementById('container')
  container.classList.add('basic_style')
  </script>
</body>
</html>

```

前者每次单独操作，都去触发一次渲染树更改，从而导致相应的回流与重绘过程。

合并之后，等于我们将所有的更改一次性发出，用一个 style 请求解决掉了。

### 将 DOM “离线”

我们上文所说的回流和重绘，都是在“该元素位于页面上”的前提下会发生的。一旦我们给元素设置 display: none，将其从页面上“拿掉”，那么我们的后续操作，将无法触发回流与重绘——这个将元素“拿掉”的操作，就叫做 DOM 离线化。

仍以我们上文的代码片段为例：

```
const container = document.getElementById('container')
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'
...（省略了许多类似的后续操作）

```

离线化后就是这样：

```
let container = document.getElementById('container')
container.style.display = 'none'
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'
...（省略了许多类似的后续操作）
container.style.display = 'block'

```

有的同学会问，拿掉一个元素再把它放回去，这不也会触发一次昂贵的回流吗？这话不假，但我们把它拿下来了，后续不管我操作这个元素多少次，每一步的操作成本都会非常低。当我们只需要进行很少的 DOM 操作时，DOM 离线化的优越性确实不太明显。一旦操作频繁起来，这“拿掉”和“放回”的开销都将会是非常值得的。

## Flush 队列：浏览器并没有那么简单

以我们现在的知识基础，理解上面的优化操作并不难。那么现在我问大家一个问题：

```
let container = document.getElementById('container')
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'

```

这段代码里，浏览器进行了多少次的回流或重绘呢？

“width、height、border是几何属性，各触发一次回流；color只造成外观的变化，会触发一次重绘。”——如果你立刻这么想了，说明你是个能力不错的同学，认真阅读了前面的内容。那么我们现在立刻跑一跑这段代码，看看浏览器怎么说：

![](https://user-gold-cdn.xitu.io/2018/10/4/1663f57519a785ab?w=1284&h=96&f=png&s=18506)

这里为大家截取有“Layout”和“Paint”出镜的片段（这个图是通过 Chrome 的 Performance 面板得到的，后面会教大家用这个东西）。我们看到浏览器只进行了一次回流和一次重绘——和我们想的不一样啊，为啥呢？

因为现代浏览器是很聪明的。浏览器自己也清楚，如果每次 DOM 操作都即时地反馈一次回流或重绘，那么性能上来说是扛不住的。于是它自己缓存了一个 flush 队列，把我们触发的回流与重绘任务都塞进去，待到队列里的任务多起来、或者达到了一定的时间间隔，或者“不得已”的时候，再将这些任务一口气出队。因此我们看到，上面就算我们进行了 4 次 DOM 更改，也只触发了一次 Layout 和一次 Paint。

大家这里尤其小心这个“不得已”的时候。前面我们在介绍回流的“导火索”的时候，提到过有一类属性很特别，它们有很强的“即时性”。当我们访问这些属性时，浏览器会为了获得此时此刻的、最准确的属性值，而提前将 flush 队列的任务出队——这就是所谓的“不得已”时刻。具体是哪些属性值，我们已经在“最容易被忽略的操作”这个小模块介绍过了，此处不再赘述。

## 小结

整个一节读下来，可能会有同学感到疑惑：既然浏览器已经为我们做了批处理优化，为什么我们还要自己操心这么多事情呢？今天避免这个明天避免那个，多麻烦！

问题在于，**并不是所有的浏览器都是聪明的**。我们刚刚的性能图表，是 Chrome 的开发者工具呈现给我们的。Chrome 里行得通的东西，到了别处（比如 IE）就不一定行得通了。而我们并不知道用户会使用什么样的浏览器。如果不手动做优化，那么一个页面在不同的环境下就会呈现不同的性能效果，这对我们、对用户都是不利的。因此，养成良好的编码习惯、从根源上解决问题，仍然是最周全的方法。