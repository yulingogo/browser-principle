#### DOM

从网络传给渲染引擎的`HTML`文件字节流是无法直接被渲染引擎理解的。所以要将其转换成渲染引擎能够理解的内部结构。这个结构是`DOM`。`DOM`提供了对HTML文档结构化的描述。在渲染引擎中，`DOM`有三个层面的作用。

+ 从页面视角，`DOM`是生成页面的基础数据结构。
+ 从`JavaScript`脚本视角看，DOM提供了`JavaScript`脚本操作的接口。从这套接口，`js`可以对`com`结构进行访问，从而改变文档的结构，样式和内容。
+ 从安全视角，`DOM`是一道安全防护线，一些不安全的内容·在`DOM`解析阶段就被拒之门外。

简言之，`DOM`是表述`HTML`的内部数据结构，它会将`Web`页面和`js`脚本连接起来，并过滤一些不安全的内容。

#### HTML什么时候解析数据

`HTML`解析器并不是等整个文档加载完成之后再开始解析，而是网络进程加载了多少数据，`HTML`解析器便解析多少数据。

#### HTML解析过程

在渲染引擎内部，有一个叫做HTML解析器（`HTMLParser`）的模块，它的作用便是将`HTML`解析为`DOM`结构。

网络进程接收到响应头后，会根据响应头中的`content-type`字段来判断文件的类型，比如`content-type`的值是`text/html`，那么浏览器就会判断这是一个`html`类型的文件。

然后为该文件选择或者创建一个渲染进程。渲染进程准备好之后，网络进程和渲染进程之间会建立一个共享数据的管道，网络进程接收到数据之后就会忘这个管道放，而渲染进程则从管道的另一端不断地读取数据，并同时将读取的数据喂给HTML解析器。

你可以将这个管道想象成一个水管，网络进程进程接收到数据像水一样倒进水管，而水管的另一端是渲染进程的`HTML`解析器，他会动态的接收字节流，并将其解析成`DOM`。

那么，字节流是如何转换成`DOM`的呢。

![img](https://static001.geekbang.org/resource/image/1b/8c/1bfcd419acf6402c20ffc1a5b1909d8c.png)

1. 第一个阶段，通过分词器将字节流转换成Token。

   与`V8`引擎解析`js`一样，解析`html`第一步也是一样。通过分词器将字节流转换成一个个`Token`，分为`Tag Token`和文本`Token`。

![img](https://static001.geekbang.org/resource/image/b1/ac/b16d2fbb77e12e376ac0d7edec20ceac.png)

​		`Tag Token`又分为`StartTag`和`EndTag`。

2. 后续第二和第三个阶段是同步进行的。需要将Token解析为`DOM`节点，并将`DOM`节点添加到`DOM`树中。
   + `HTML`解析器维护了一个`Token`栈结构，该栈主要用来计算节点之间的父子关系，在第一个阶段生成的`Token`会顺序压入这个栈中
   + 如果压入栈中的Token是一个`StartTag Token`，HTML解析器会为该Token结构创建一个`DOM`节点，并将该节点加到`DOM`树中，他的父节点就是栈中相邻那个元素生成的节点。
   + 如果压入栈中的是一个文本`Token`，那么生成一个文本节点，然后将该节点加到`DOM`树中，文本`Token`不需要入栈，他的父节点就是当前栈顶`Token`对应的那个节点。
   + 如果压入栈中的是一个`EndTag Token`，比如是`EndTag div`，那么HTML解析器会查看当前栈顶是不是`StartTag div`。如果是，则`StartTag div`出栈，该div元素解析完成。

通过分词器解析的`Token`就是这样不停的压栈和出栈，整个解析过程就是这样不停的解析下去，直到分词器将所有的字节流分词完成。

![img](https://static001.geekbang.org/resource/image/7a/f1/7a6cd022bd51a3f274cd994b1398bef1.png)

默认有一个根为`document`的结构。

#### js如何影响dom树构建

```javascript

<html>
<body>
    <div>1</div>
    <script>
    let div1 = document.getElementsByTagName('div')[0]
    div1.innerText = 'time.geekbang'
    </script>
    <div>test</div>
</body>
</html>
```

之前流程一样。解析到`script`标签时，渲染引擎判断这是一段脚本，此时`HTML`解析器就会暂停`DOM`的解析，因为接下来的`javascript`可能改变当前的`dom`结构。

这时`HTML`解析器暂停工作，`javascript`引擎介入，并执行`script`标签内的这段脚本。脚本执行完毕，`HTML`解析器恢复解析过程，继续解析后面的内容。

这个是比较简单的，接下来是通过`script`引入脚本。

具体过程与上述一致，但是会有一个下载过程。`javascript`文件的下载过程会阻塞DOM解析。而通常下载是很费时间的，会受到网络环境，js大小等因素的影响。

`Chrome`浏览器做了很多优化。其中一个主要的优化是预解析操作。

当渲染引擎收到字节流之后，会开启一个预解析线程，用来分析 HTML 文件中包含的 JavaScript、CSS 等相关文件，解析到相关文件之后，预解析线程会提前下载这些文件。

我们知道`javascript`线程会阻塞`DOM`，解决办法，比如使用CDN来加速`javascript`的加载，压缩`js`的体积等。另外，如果`javascript`中没有操作`dom`的代码，可以设置异步加载，通过`async`和`defer`来标记代码。

`async`加载完立即执行

`defer`需要等到`DomContentLoaded`事件之前执行。



第三种情况，外部脚本js还引用了css文件。

在执行js之前，需要等待外部的css文件下载完成，并解析生成cssom对象之后，才能执行js脚本。

而js引擎在解析js之前，是不知道js是否操纵了cssom的，所以渲染引擎在遇到js脚本时，不管该脚本是否操纵了cssom，都会执行css文件下载解析操作，在执行js脚本。

