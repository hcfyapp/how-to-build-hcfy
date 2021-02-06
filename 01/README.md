# 检测用户的划词操作

我会在文章的末尾将文章中写过的代码打包成一个扩展程序并在浏览器中运行，目前，我们先探讨一下如何使用前端技术来检测用户的划词操作。

## 用鼠标划词的情况

选中一段文本最常见的方式就是用鼠标双击、三击或在页面上拖动。对于这种情况，监听页面的 `mouseup` 事件就可以检测到了，示例代码如下：

```js
document.addEventListener('mouseup', () => {
  // 获取页面中选中的文本。这是最常用的一种方式，但并不能覆盖到所有情况，下一篇文章中会详细介绍。
  const text = window.getSelection().toString()
  // 如果页面中没有选中的文本，那么什么都不做
  if(!text) return
  console.log(text)
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
    console.log(text)
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

1. 划选一段文本，例如“就划选双引号里面的这段吧”。
2. 点击你划选的这段文本。

第一步划选文本之后，控制台会正常打印出来你划选的文本，这是符合我们的预期的。在第二步中，点击选中文本之后，选中的文本就消失了，但是，此时控制台仍然打印出了你之前划选的文本——这就不符合我们的预期了。

如果你没对这种情况做处理，那么就会出现用户点击了他划选的文本后，页面上划选文本消失了，可是我们却显示出来了翻译按钮情况。

`setTimeout` 解决了这个问题，你可以粘贴下面这段代码试一下：

```js
document.addEventListener('mouseup', () => window.setTimeout(() => console.log(window.getSelection().toString())))
```

有些用户会在触摸屏上使用我们的扩展程序（例如 Microsoft Surface，这是[真实存在的案例](https://github.com/lmk123/crx-selection-translate/issues/412)），你还可以考虑加上对 touch 事件的支持，不过这就不在我们的讨论范围之内了。

## 鼠标划词之外的情况

在页面上产生选中的文本并不是只有用鼠标这一种情况，一些快捷键也可以选中文本，例如 Ctrl + A；另外，Ctrl + Z 也会还原在文本框中之前选中的文本。不同的浏览器、操作系统的快捷键不尽相同，所以靠枚举快捷键来检测选中文本的操作也不靠谱。最理想的办法是，有没有一种跟用户操作无关的检测方式，无论页面中选中的文本是怎么产生的，我们都能得到通知。

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

虽然划词翻译没有用到这个方案，但并不代表这个方案没有作用，它只是在划词翻译里不适用而已，换一个场景可能就是绝佳的解决方案了。

## 将代码打包成扩展程序并运行在浏览器中

到目前为止，我们已经有了一个用于检测用户划词操作的 js 文件，现在，我们就来将这个文件打包成一个扩展程序并运行在浏览器中。

<details>
<summary>展开阅读</summary>

TODO

</details>

## 总结

感谢你阅读到了这里！在这篇文章里，我介绍了这些内容：

- 内容脚本简介。
- 用户划词操作的检测。
- 打包扩展程序。

如果你有疑问，欢迎给我提 issue。
