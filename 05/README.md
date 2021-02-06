
TODO: 将跨 iframe 显示翻译面板的内容转移到第五篇中来。以下是原本在第一篇中的内容。

## 跨 iframe 的划词操作检测

当我们开发自己的网站时，我们只需要关注自己网站的结构就可以了，但作为扩展程序时，我们的代码会被注入到互联网里的任何网站里，这些网站使用的技术五花八门，我们的扩展程序可能在大部分网站里都能正常工作，但偶尔就会在一些特定的网站出现问题；反过来，我们的扩展程序也可能会导致一些网站不能正常工作。

后面我会专门用一篇文章介绍如何尽可能的避免我们的扩展程序和用户的网页产生冲突，在这篇文章里，我先针对在有 iframe 的页面里进行划词检测的情况进行介绍。

扩展程序支持将我们的代码注入到标签页的每一个窗口中运行，既包括顶层窗口（topmost window，即 [`window.top`](https://developer.mozilla.org/en-US/docs/Web/API/Window/top)），也包括 iframe 或 frame 元素里的窗口。

划词翻译一开始就用了这种方式，这样一来，每个窗口里都有各自的翻译按钮和翻译结果，同一个标签页的多个窗口之间相互独立，互不干扰。但是，这种简单的处理方式暴露出了这些问题：

- 翻译结果的弹窗被限制在了 iframe 的可视区域里，不能完整显示，见 [lmk123/crx-selection-translate#834](https://github.com/lmk123/crx-selection-translate/issues/834)。
- 翻译结果的弹窗被顶层窗口的元素遮挡了，见 [lmk123/crx-selection-translate#840](https://github.com/lmk123/crx-selection-translate/issues/840)。
- 有的 iframe 宽度很小，导致被翻译结果弹窗撑开了横向或纵向滚动条，见 [lmk123/crx-selection-translate#873](https://github.com/lmk123/crx-selection-translate/issues/873)。

解决这些问题的办法只有一个，就是只在顶层窗口显示翻译结果弹窗。

这种情况下，我们就需要将我们的代码一分为二了：

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
        console.log(text)
        // 执行后续操作，例如显示翻译按钮
      }
      break
  }
})
```

### 一个兼容性问题

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
        console.log(text)
        // ...
      }
      break
  }
})
```

这样就可以同时兼容 Chrome 和 Firefox 了。
