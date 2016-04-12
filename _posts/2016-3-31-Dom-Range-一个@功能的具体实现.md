---
keywords: range, rangy, @功能, At功能, contenteditable
description: Dom Range的相关介绍，并以此实现一个带@选人功能的编辑框
---
{% if page.description %}
>{{ page.description }}
{% endif %}

# 前言
浏览网页时，偶尔能看到选中某段文字能够进行分享或者按钮，这个就是用Range配合Selection来实现的，Selection即为当前选区，转换为Range对象就能对选区范围内的片段进行一系列操作，比如获取选区内的文本内容等等。

# Range及其使用
简单来说，Range是html中的任意一段内容（fragment），用这玩意儿，可以让你来得到html中的任意部分。

Range有几个基本的属性来反应当前内容的范围，见下图。

<img alt="Dom Range" src="https://gw.alicdn.com/tps/TB18RP4MXXXXXXhXXXXXXXXXXXX-872-722.png" width="436" />

startOffset和endOffset从0开始，包含startOffset ，不包含 endOffset。

Range存在w3c range（我是标准）、[Microsoft TextRange](https://msdn.microsoft.com/en-us/library/ms535872(VS.85).aspx)（一看就知道是IE专用，ie8及以下），Selection和Range兼容相关问题这里不做介绍了。那么先来简单熟悉下标准api。

## 获取range对象

```js
// 从selection获取range 
var selection = window.getSelection();
var range = selection.getRangeAt(0); 
// 创建range
var range = document.createRange(); 
``` 

## 设置range范围

```js
// 设置起止位置 另外还有setStartBefore/setStartAfter和setEndBefore/setEndAfter
range.setStart(startContainer, startOffset);
range.setEnd(EndContainer, endOffset);
// 设置边界为referenceNode的起止位置（包含referenceNode）
range.selectNode(referenceNode);
// 设置边界为referenceNode的起止位置（不包含referenceNode）
range.selectNodeContents(referenceNode);
// 将边界进行折叠 toStart参数表示是否向前（起始边界）折叠
range.collapse(toStart);
``` 

## 处理range内容(获取、删除、插入) 

```js
// 抽取html内容为fragment-将从页面中移除该范围内的片段
range.extractContents();
// 获取文本内容
range.toString()
// 删除内容
range.deleteContents();
// 在range起始位置插入一个node
var textNode = document.createTextNode('new node');
range.insertNode(textNode);
``` 

具体的api描述可见[MDN](https://developer.mozilla.org/en-US/docs/Web/API/range)，下面基于[rangy](https://github.com/timdown/rangy)库进行开发，这个库主要处理了浏览器的兼容问题（ie lt9），并且提供了额外的api和各类插件方便开发（虽然在这里没有用到，而实际上自行实现几个核心api兼容即可满足这里的需求）。

# @功能的实现
@功能大致可分为两种情形，一种是实时输入，在光标右下方弹出搜索结果面板，另一种是在打开新的选择页，在其中完成选择。两种效果图如下：

<img alt="Dom Range" src="https://gw.alicdn.com/tps/TB1CwaKMpXXXXaPXVXXXXXXXXXX-860-880.png" width="430" />

看到效果图应该能发现，在编辑区（contenteditable）插入独立的标签就是range的功劳了，当然在插入标签前后的光标位置也需要range配合selection进行控制。

这里实现的是第二种（  先来个 **[demo](http://jzlingmo.github.io/rc-mention/)** 体验一下，为啥有v2？详情请见下文），主要有三个阶段：

1. **触发选人** -  两种方式，输入@触发和点击按钮触发

2. **进行选人** - 调用其它组件完成，返回选择人物列表

3. **完成选人** - 找到插入点，插入人物标签

阶段2由于是调用外部组件，忽略。另外两个阶段的关联之处就是人物标签的插入点（bookmark），那么关于bookmark的改变就显得比较关键了。

实际上，如果不去考虑按钮触发的插入也需要保证在之前的光标位置后（统一从末尾插入），那就远不用这么麻烦了。但是作为攻(cheng)城(xu)狮(yuan)我们要有点追求，讲究挖坑、踩坑、填坑。

一开始考虑的保存bookmark的时机有两个，输入@保存bookmark和 失去焦点保存bookmark，还需要能够判断bookmark是否有效（因为可以被delete掉），而在后面直到发现blur时保存bookmark出现问题才重新去思考另一种方式（v2），当然v1也是可以实现，我们先一路走到头深入接触下range。

![](https://gw.alicdn.com/tps/TB1CffYMXXXXXXSXpXXXXXXXXXX-491-268.svg)


## 触发选人阶段

### 输入@触发

```js
emitChange(){
	// 监听用户输入
	let t = this;
	let sel = rangy.getSelection();
	let range = sel.getRangeAt(0);
	if (range.commonAncestorContainer.nodeType === Node.TEXT_NODE) {
		range.setStart(range.commonAncestorContainer, 0);
		let originStr = range.toString();
		if (originStr.substr(-1, 1) === '@') {
			 // 重置range将@字符包含在内
			 range.setStart(range.commonAncestorContainer, originStr.length - 1);
			 // 选人前保存书签 - rangy提供的非标准api 下一节有介绍
			t._bookmark = range.getBookmark(range.commonAncestorContainer);
			t.onMention(); // 打开选人对话框
		}
	}
}
``` 

### 点击按钮触发
这个情况完成选人后用的bookmark是在 blur时候设置的，如果初始情况是这样的：

“一二三[标签]”

用^代表bookmark ，应是：

“一二三[标签]^”

但实际上，由于这里使用的_rangy_库里的_getBookmark_方法比想象中的功能弱了些，只适用于文本节点保存位置，实际上该方法主要是用于文本内容不变但html结构发生变化的情况（比如富文本编辑器的加粗等）。如果使用该方法得到的结果是：

“一二三^[标签] ”

那么点击3次按钮进行插入的结果就是：

"一二三^ [新标签3][新标签2] [新标签1][标签]"


**一种简单的处理方法**

输入@标签后的**range.commonAncestorContainer**是**TextNode**，而在input标签后的则是editor节点，是**ElementNode**，那么在**完成选人部分**加个判断，是ElementNode都统一定位到末尾进行插入（入坑无法自拔）。

以下是一个实用的**移动光标到元素末尾**的方法。

```js
_setCaretToEnd(editor){
	let selection = rangy.getSelection();
	let range = rangy.createRange();
	range.selectNodeContents(editor);
	range.collapse(false);
	range.select();
	// 上一行的select是rangy提供的非标准api，可用以下两行代替
	//selection.removeAllRanges();
	//selection.addRange(range);
	return range
}
```

**另一种我们自定义方法完成bookmark的设置，在下文中再单独来描述。**

## 完成选人阶段

### 在指定的位置插入选中人的标签

```js
 handleMentionAdd(persons) {
	let t = this;
	let editor = this.refs.editor.getDOMNode();
	let range = rangy.createRange();

	// 移动到插入位置
	if (t._bookmark.containerNode.nodeType === Node.TEXT_NODE) {
		range.moveToBookmark(t._bookmark);
	} else {
		range = this._setCaretToEnd(editor);
	}
	// 创建即将插入的html片段
	let mentionFragment = t._createMentionNode(persons);
	range.deleteContents(); // 清除选中内容-@符号
	// 插入片段前先保存末尾节点供后续重置range
	let lastChild = mentionFragment.lastChild;
	range.insertNode(mentionFragment);
	// 重置range到新插入的标签之后 并显式地移动光标
	range.setStartAfter(lastChild, 0);
	range.collapse(true);
	range.select();
	t._bookmark = range.getBookmark(range.commonAncestorContainer);
	t.emitChange();
}
``` 

### 返回格式化后的文本

主要是递归遍历每个节点，只提取文本、input和br，进行处理后拼接得到最后的文本。

```js
for (let i = 0, len = nodes.length; i < len; i += 1) {
	let node = nodes[i];
	if (node.nodeType === Node.ELEMENT_NODE) {
		let tagName = node.tagName.toLowerCase();
		if (tagName === 'input') { // input element
			let item = JSON.parse(node.getAttribute('data'));
			// 这里格式化标签的取值
			content += t.props.formatValue(item);
		} else if (tagName === 'br') {
			content += '\n';
		}else{
			// 递归调用 遍历子节点
			content += t.extractContents(node)
		}
	} else if (node.nodeType === Node.TEXT_NODE) {
		content += node.nodeValue;
	}
}
``` 

## 自定义bookmark的创建和定位

### 创建bookmark
主要思路是通过新增两个隐藏的标记点来设置bookmark。

serializable为true则保存的是节点id，主要是为了能够根据id判断bookmark是否存在（要是被用户删掉了就找不到插入点了）。

```js
createBookmark(range, serializable){
	var startNode, endNode, baseId, clone;
	var collapsed = range.collapsed;
	var prefix = '__bookmark__';
	startNode = document.createElement('span');
	startNode.style.display = 'none';
	if (serializable) {
		baseId = prefix + (this._bookmarkId++);
		startNode.setAttribute('id', baseId + ( collapsed ? 'C' : 'S' ));
	}
	if (!collapsed) {
		endNode = startNode.cloneNode(true);
		if (serializable) {
			endNode.setAttribute('id', baseId + 'E');
		}
		// 在终点插入标记
		clone = range.cloneRange();
		clone.collapse(false);
		clone.insertNode(endNode);
	}
	// 在起点插入标记
	clone = range.cloneRange();
	clone.collapse(true);
	clone.insertNode(startNode);
	if (endNode) {
		range.setEndBefore(endNode);
	}
	range.setStartAfter(startNode);
	return {
		startNode: serializable ? baseId + ( collapsed ? 'C' : 'S' ) : startNode,
		endNode: serializable ? baseId + 'E' : endNode,
		serializable: serializable,
		collapsed: collapsed
	}
}
```

### 定位到bookmark

```js
moveToBookmark(bookmark, range){
	range = range || document.createRange();
	var serializable = bookmark.serializable,
	startNode = serializable ? document.getElementById(bookmark.startNode) : bookmark.startNode,
	endNode = serializable ? document.getElementById(bookmark.endNode) : bookmark.endNode;
	// 设置起止位置并移除标记点
	range.setStartAfter(startNode);
	startNode.remove();
	if (endNode) {
		range.setEndBefore(endNode);
		endNode.remove();
	} else {
		range.collapse(true);
	}
	return range
}
``` 

### 进行调用
```js
let selection = rangy.getSelection();
let range = selection.getRangeAt(0);
// 创建bookmark
let bookmark = rangy. createBookmark(range, true);
// ... 完成选人之后需要进行插入标签
// 移动到bookmark位置
rangy.moveToBookmark(bookmark);
```

# 实现过程中的一些问题
- contenteditable元素无法focus

body设置了-webkit-user-select:none导致无法focus，于是在该元素上重设为-webkit-user-select:text

- 在手机（以下均指iPhone6的webview及safari）上点击编辑框focus较慢（需要深度点击）

系统升级后该问题不存在了，对于有该问题的版本，可加上onClick事件来触发focus事件（非focus状态）。另外触发focus将光标置到开头的话只要调用上面提到的setCaretToEnd方法挪到最后就行了。

- enter换行会得到两个br标签，继续输入才会消除第二个br

这里在格式化输入时对br标签进行处理，删除多余的换行

`content.replace(/\n$/, '').replace(/\n\n/g, '\n') `

并且当格式化后的content为空时做一次清空处理，清除遗留的一个br

- 在手机上blur事件处理中通过Selection获取range对象阻塞blur事件的后续执行

经过调试发现safari在编辑区blur后Selection对象的rangeCount为0（chrome中blur后仍保留之前的Selection），你可以打开任意页面选中文字后点击其它地方触发blur，可以看到safari中的选区在blur后会消失，而chrome的选区只是背景色变灰了，所以调用getRangeAt(0)会出现报错。

对于该问题，可改变bookmark记录的时机，光标位置通过点击、输入或者方向键来改变（手机端还有touch事件），因此可以考虑在这些情况下就更新bookmark。

- 关于v2

到这里突然发现已经不用考虑bookmark被删除的情况了，因为删除操作（keyup）后也会重新保存bookmark，于是干脆完结[v1](https://github.com/jzlingmo/rc-mention/tree/v1)，产出[v2](https://github.com/jzlingmo/rc-mention/tree/v2)。

与v1不同的是，v2中的bookmark只是单纯地保存了range，并且每个触发光标移动的情况（搜狗输入法等能够在虚拟键盘滑动来控制光标那也是没办法了）下都会设置一次，大部分的逻辑都在keyup事件中，具体可查看 **[相关代码](https://github.com/jzlingmo/rc-mention/blob/v2/src/MentionEditor.js#L94)**。

# 移动端的使用
rangy库中的core文件大小为150k+，引入rangy库在压缩打包后会带来47k左右的体积增加，考虑到移动端的浏览器兼容性要好于pc端（再见，ie），上述提到的标准api都可以进行调用，只需要对部分rangy的api做相应替换即可，主要有以下几个：

```js
// 1. rangy.getSelection 获取Selection对象
let selection = window.getSelection();
// 2. rangy.createRange 创建Range对象 也可用selection.getRangeAt(0)获取
let range = document.createRange();
// 3. range.select 用于显式地设置选区位置
selection.removeAllRanges();
selection.addRange(range);
```

这些处理放在了[rangy同名文件](https://github.com/jzlingmo/rc-mention/blob/v2/src/rangy.js)中，如果在之前引入了rangy库则会将其取代。

至于v1的bookmark，我们在上述已经以工具方法的形式进行提供了，那么在定位bookmark时就不用判断commonAncestorContainer了，只要判断标记节点是否存在就可以了。

``` js
if(t._bookmark && document.getElementById(t._bookmark.startNode)){
	range = rangy.moveToBookmark(t._bookmark, range);
}else{
	range = this._setCaretToEnd(editor);
}
``` 

当然在v2中bookmark已经没有存在的必要了，一切又回归到最初，入坑到出坑，灵光乍现。

最后是完整项目代码链接：
 
**[v1入口](https://github.com/jzlingmo/rc-mention/tree/v1)** 和 **[v2入口](https://github.com/jzlingmo/rc-mention/tree/v2)**

# 总结
初看Range感觉比较冷门，细想一下，在编辑器中还是有比较丰富的应用的，比如[ckeditor](https://github.com/ckeditor/ckeditor-dev/)就是基于Range进行实现的。入坑后，至此也是对range有较深印象了，对于更为复杂的功能实现，不妨可以使用或者借鉴一下rangy的一些扩展。

# 参考资料

>[https://dom.spec.whatwg.org/#introduction-to-dom-ranges](https://dom.spec.whatwg.org/#introduction-to-dom-ranges)

> [https://developer.mozilla.org/en-US/docs/Web/API/range](https://developer.mozilla.org/en-US/docs/Web/API/range)

> [https://github.com/timdown/rangy/wiki/Rangy-Range](https://github.com/timdown/rangy/wiki/Rangy-Range)
