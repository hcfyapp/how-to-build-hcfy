# 检测用户的划词操作

划词翻译的第一步就是检测用户的划词操作。为了能在任意页面检测到用户的划词操作，我们需要在每个页面都注入一段检测代码，这时候就需要用到扩展程序提供的内容脚本功能了。

## 内容脚本简介

内容脚本（Content scripts）是扩展程序最重要的一个组成部分，因为大部分扩展程序都需要对用户打开的页面做一些监听或修改以实现扩展的功能，很多扩展程序甚至只需要内容脚本就可以完成它们的功能，网络上很流行的[油猴脚本](https://greasyfork.org/zh-CN/scripts)就属于这一类。

> 划词翻译的第一个版本就只用到了内容脚本，但后面的文章会解释为什么只用内容脚本是不够的。

Chrome 和 Firefox 都对内容脚本进行了专门的介绍，有兴趣的读者可以先看一遍：

- [Content scripts - Chrome Developers](https://developer.chrome.com/docs/extensions/mv2/content_scripts/)
- [Content scripts - Mozilla | MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Content_scripts)

如果你懒得看，那么你只需要记住这点就可以无障碍的阅读后面的内容了：**内容脚本运行在用户打开的网页里，相当于浏览器帮你插入了一个指向你扩展程序内的 js 文件的 `<script>` 元素**。

> 当然，内容脚本跟普通的 `<script>` 肯定是有不同之处的，但这篇文章并没有涉及到这些不同，感兴趣的读者可以阅读上面的官方文档。

## 用鼠标划词的情况

选中一段文本最常见的方式就是用鼠标双击、三击或在页面上拖动。对于这种情况，监听页面的 `mouseup` 事件就可以检测到了，示例代码如下：

```js
document.addEventListener('mouseup', () => {
  // 获取页面中选中的文本。这是最常用的一种方式，但并不能覆盖到所有情况，下一篇文章中会详细介绍。
  const text = window.getSelection().toString()
  // 如果页面中没有选中的文本，那么什么都不做
  if(!text) return
  // 如果有，那么执行下一步操作，例如显示一个翻译按钮，或者直接弹出翻译窗口
})
```

全文完。

……等等先别关，事情并没有这么简单。

这段代码少了一个步骤：**我们需要使用 `setTimeout()` 包裹里面的代码**。

```js
document.addEventListener('mouseup', () => {
  // 使用 setTimeout() 包裹下面的代码
  window.setTimeout(() => {
    const text = window.getSelection().toString()
    if(!text) return
    // ...
  })
})
```

`setTimeout` 解决了一个比较特殊的问题：当用户正好点击了他之前划选的文本的情况。

Talk is cheap. Show me the code.

你现在应该正好在浏览器上阅读这篇文章。你可以打开浏览器的控制台，粘贴下面的代码：

```js
document.addEventListener('mouseup', () => console.log(window.getSelection().toString()))
```

然后执行以下步骤：

1. 划选一段文本，例如“就划选双引号里面的这段吧”。此时控制台会正常打印出来你划选的文本，这是符合我们的预期的。
2. 点击你划选的这段文本。

点击之后，页面上选中的文本就消失了，但是，此时控制台仍然打印出了你之前划选的文本——这就不符合我们的预期了。

如果你没对这种情况做处理，那么就会出现用户点击了他划选的文本后，页面上划选文本消失了，可是我们却显示出来了翻译按钮情况。

`setTimeout` 解决了这个问题，你可以粘贴下面这段代码试一下：

```js
document.addEventListener('mouseup', () => window.setTimeout(() => console.log(window.getSelection().toString())))
```

有些用户会在触摸屏上使用我们的扩展程序（例如 Microsoft Surface，这是[真实存在的案例](https://github.com/lmk123/crx-selection-translate/issues/412)），你还可以考虑加上对 touch 事件的支持，这就不在我们的讨论范围之内了。

## 鼠标划词之外的情况

在页面上产生选中的文本并不是只有用鼠标这一种情况，例如，用户可以点击一个文本框，输入一些文字后，用快捷键 Ctrl + A 全选里面的文本；另外，Ctrl + Z 也会还原在文本框中之前选中的文本，所以单单只检测 `Ctrl + A` 也不靠谱。最理想的办法是，有没有一种跟用户操作无关的检测方式，无论页面中选中的文本是怎么产生的，我们都能得到通知。

而事实上，浏览器也确实提供了这样的检测方式：[`selectionchange` 事件](https://developer.mozilla.org/en-US/docs/Web/API/Document/selectionchange_event)。你可以在浏览器控制台中输入以下代码，然后在页面上用鼠标或 Ctrl + A 选中文本试试：

```js
document.addEventListener('selectionchange', () => console.log(window.getSelection().toString()))
```

无论文本是如何被选中的，控制台都会将页面中选中的文本打印出来，但这个事件也有一个问题：它触发的太密集了。

当我们用鼠标划词的时候，鼠标按下去的时候会触发一次，控制台会打印一个空字符串；之后拖动鼠标时，每选中一个文字都会触发一次。然而，浏览器虽然提供了 [`selectstart` 事件](https://developer.mozilla.org/en-US/docs/Web/API/Document/selectstart_event)，却没有提供 `selectend` 事件，所以如果要使用它，我们需要自行判断划词结束的时机。

从技术上也是可以解决这个问题的。例如，把 `selectionchange` 事件跟鼠标事件结合起来使用，如果 `selectionchange` 事件是在 `mousedown` 事件之后产生的，那么就用 `mouseup` 事件作为划词结束的时机，否则就用 [debounce() 方法](https://lodash.com/docs/4.17.15#debounce)来检测。

但是，我在开发划词翻译的时候没有使用这个方案，而是提供了辅助键的功能，在用户按下 Ctrl 键的时候读取页面上选中的文本并翻译。我的理由如下：

- 鼠标划词外的情况大多出现在文本编辑的时候，这种时候，用户一般并不希望弹出翻译按钮。辅助键正好覆盖了这种需要用户确认（即按一下按钮）才进行翻译的情况。
- 以上一个理由作为前提，`selectionchange` 比 `mouseup` 的方案需要更多的代码，这意味着出现问题的可能性也更大了，而我希望划词翻译的代码和功能都尽可能的简单小巧。

虽然划词翻译没有用到这个方案，但并不代表这个方案没有作用，它只是在划词翻译里不适用而已，换一个场景可能就是绝佳的解决方案了。另一方面，接下来要介绍的跨 iframe 检测跟这一小节的内容也有点关系。

## 跨 iframe 的划词操作检测

当我们开发自己的网站时，我们只需要关注自己网站的结构就可以了，但内容脚本可能会被注入到互联网里的任何网站里，这些网站使用的技术五花八门，我们的扩展程序可能在大部分网站里都能正常工作，但偶尔就会在一些特定的网站出现问题；反过来，我们的扩展程序也可能会导致一些网站不能正常工作。

后面我会专门用一篇文章介绍如何尽可能的避免我们的扩展程序和用户的网页产生冲突，在这篇文章里，我先针对在有 iframe 的页面里进行划词检测的情况进行介绍。

内容脚本有一个 `"all_frames": true` 的选项，意思是将我们声明的内容脚本注入到标签页的每一个窗口中，既包括顶层窗口（topmost window，即 [`window.top`](https://developer.mozilla.org/en-US/docs/Web/API/Window/top)），也包括 iframe 或 frame 元素里的窗口。

划词翻译一开始就用了这种方式覆盖了在 iframe 中划词的场景，但是，这种简单的处理方式暴露出了这些问题：

- 翻译结果的弹窗被限制在了 iframe 的可视区域里，不能完整显示，见 [lmk123/crx-selection-translate#834](https://github.com/lmk123/crx-selection-translate/issues/834)。
- 翻译结果的弹窗被顶层窗口的元素遮挡了，见 [lmk123/crx-selection-translate#840](https://github.com/lmk123/crx-selection-translate/issues/840)。
- 有的 iframe 宽度很小，导致被翻译结果弹窗撑开了横向或纵向滚动条，见 [lmk123/crx-selection-translate#873](https://github.com/lmk123/crx-selection-translate/issues/873)。

这些问题出现的原因都是同一个，即每个 iframe 里都有一个翻译结果弹窗，解决的办法也是同一个，也就是只在顶层窗口显示翻译结果弹窗。

这种情况下，我们就需要将内容脚本一分为二了：

- 一部分代码注入到所有窗口中（当然，也包括顶层窗口自己），专门用来检测鼠标划词、按下辅助键等用户操作，并通知顶层窗口。
- 另一部分代码只注入到顶层窗口中，用于接收通知并做出反应，例如显示翻译结果弹窗。

我们可以使用 `window.postMessage()` 在窗口之间进行消息通信，示例代码如下。

首先，我们将检测用户操作的文件命名为 `content-script-in-all-frames.js`，代码如下：

```js
document.addEventListener('mouseup', () => {
  window.setTimeout(() => {
    window.top.postMessage({
      action: '用户用鼠标划词了',
      text: window.getSelection().toString()
    }, '*')
  })
})
```

然后，我们将只注入到顶层窗口中的文件命名为 `content-script-in-topmost-window.js`，代码如下：

```js
window.addEventListener('message', ({data}) => {
  // 用户浏览的网页自己也可能会发送消息过来，所以需要进行过滤
  switch (data?.action) {
    case '用户用鼠标划词了':
      const text = data.text
      if(text) {
        // 执行后续操作，例如显示翻译按钮
      }
      break
  }
})
```

这段代码在 Chrome 中运行正常，但是在 Firefox 中就会报错——恭喜你，你遇到了第一个属于扩展程序的“兼容性问题”。

在 Firefox 中，顶层窗口接收到的对象如果来自 iframe，那么 Firefox 会将这个对象“保护”起来，你对它进行读取操作还好，但如果你试图修改它，那么就会报错，详细介绍见 [Sharing objects with page scripts - Mozilla | MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Sharing_objects_with_page_scripts)，文章里也提到了需要用 Firefox 独有的 [cloneInfo() 方法](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Language_Bindings/Components.utils.cloneInto) 对对象进行一些处理。

但是，经过我的测试，最容易实现且跨浏览器的解决方案是——使用字符串进行消息传递。我们可以在发送消息之前用 `JSON.stringify()` 方法将对象转换成字符串，然后在接收消息时尝试用 `JSON.parse()` 方法将消息转换成对象。

所以，我们的代码最终应该是这样子的：

```js
// content-script-in-all-frames.js
document.addEventListener('mouseup', () => {
  window.setTimeout(() => {
    // 将对象转成字符串
    window.top.postMessage(JSON.stringify({
      action: '用户用鼠标划词了',
      text: window.getSelection().toString()
    }), '*')
  })
})
```

```js
// content-script-in-topmost-window.js
window.addEventListener('message', ({data}) => {
  let msg
  try {
    // 将字符串转成对象
    msg = JSON.parse(data)
  } catch (e) { return }
  if (!msg) return

  switch (msg.action) {
    case '用户用鼠标划词了':
      const text = msg.text
      if(text) {
        // ...
      }
      break
  }
})
```

这样就可以同时兼容 Chrome 和 Firefox 了。

注意，在这篇文章中，我们只发送了用户选中的文本内容，但在实际情况中，我们需要发送更多信息，例如：

- 我们需要 `mouseup` 事件时鼠标指针的 X 轴和 Y 轴的位置确定翻译按钮的显示位置
- 我们需要选中文本的位置，确定翻译结果弹窗的位置。而且，这个位置并不是静态的，例如用户在 iframe 里划选了文本，但在顶层窗口滚动了网页，此时翻译结果弹窗就需要重新定位。
- 我们甚至需要将 Event 对象发送给顶层窗口，例如，如果划词操作是在顶层窗口产生的，我们需要使用 `event.target` 判断这次划词操作是否发生在我们的翻译结果弹窗里，但很显然，`window.postMessage()` 是发送不了的。

这些问题会在下一篇文章里介绍。

## 总结

感谢你阅读到了这里！在这篇文章里，我介绍了这些内容：

- 内容脚本简介。
- 用户划词操作的检测。
- 解决跨 iframe 划词的相关问题。
- 遇到并解决了第一个属于扩展程序的兼容性问题（~~就 Firefox 事多~~后面的文章中还有更多）。

如果你对这篇文章的内容有疑问，欢迎给我提 issue。
