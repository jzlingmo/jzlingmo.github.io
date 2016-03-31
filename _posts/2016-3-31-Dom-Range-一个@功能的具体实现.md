---
keywords: range, rangy, @功能, At功能, contenteditable
description: Dom Range的相关介绍，并以此实现一个带@选人功能的编辑框
---
{% if page.description %}
>{{ page.description }}
{% endif %}

## 前言
浏览网页时，偶尔能看到选中某段文字能够进行分享或者按钮，这个就是用Range配合Selection来实现的，Selection即为当前选区，转换为Range对象就能对选区范围内的片段进行一系列操作，比如获取选区内的文本内容等等。

## Range及其使用
简单来说，Range是html中的任意一段内容（fragment），用这玩意儿，可以让你来得到html中的任意部分。

Range有几个基本的属性来反应当前内容的范围，见下图。

<img alt="Dom Range" src="https://gw.alicdn.com/tps/TB18RP4MXXXXXXhXXXXXXXXXXXX-872-722.png" width="436" />

startOffset和endOffset从0开始，包含startOffset ，不包含 endOffset。

Range存在w3c range（我是标准）、Microsoft TextRange（一看就知道是IE专用，ie8及以下），Selection和Range兼容相关问题这里不做介绍了。那么先来简单熟悉下api。

### 获取range对象

```js
// 从selection获取range 
var selection = window.getSelection();
var range = selection.getRangeAt(0); 
// 创建range
var range = document.createRange(); // >=ie9
var range = new Range(); // not ie
``` 

### 设置range范围

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

### 处理range内容(获取、删除、插入) 

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

具体的api描述可见[MDN](https://developer.mozilla.org/en-US/docs/Web/API/range)，下面将使用[rangy](https://github.com/timdown/rangy)库进行开发，这个库处理了浏览器的兼容问题（主要是老版本的IE），并且提供了额外的api和各类插件方便开发。

## @功能的实现
@功能大致可分为两种情形，一种是实时输入，在光标右下方弹出结果面板，另一种是在打开新的选择页，在其中完成选择。

这里实现的是第二种，主要有三个阶段：

1. **触发选人** -  两种方式，输入@触发和点击按钮触发

2. **进行选人** - 调用其它组件完成，返回选择人物列表

3. **完成选人** - 找到插入点，插入人物标签

阶段2由于是调用外部组件，忽略。另外两个阶段的关联之处就是人物标签的插入点（bookmark），那么关于bookmark的改变就显得比较关键了。保存bookmark的时机有两个（分别用于上述两种方式的触发选人），输入@保存bookmark和 失去焦点保存bookmark。

![](https://gw.alicdn.com/tps/TB1CffYMXXXXXXSXpXXXXXXXXXX-491-268.svg)

### 触发选人阶段

#### 输入@触发

```js
emitChange(){
	// 监听用户输入
	let t = this;
	let editor = t.refs.editor;
	let len = editor.innerText.length;
	let lastLen = t.totalLen;
	t.totalLen = len;
	// 删除文本时不触发选人对话框的打开
	if (lastLen && len < lastLen) { 
		return
	}
	let sel = rangy.getSelection();
	let range = sel.getRangeAt(0);
	if (range.commonAncestorContainer.nodeType === Node.TEXT_NODE) {
		range.setStart(range.commonAncestorContainer, 0);
		let originStr = range.toString();
		if (originStr.substr(-1, 1) === '@') {
			 // 重置range将@字符包含在内
			 range.setStart(range.commonAncestorContainer, originStr.length - 1);
			 // 保存书签 rangy提供的非标准api 在下文中马上就有相关介绍
			t._bookmark = range.getBookmark(range.commonAncestorContainer);
			t.onMention(); // 打开选人对话话框
		}
	}
}
``` 

#### 点击按钮触发
这个情况完成选人后用的bookmark是在 blur时候设置的，如果初始情况是这样的：

“一二三[标签]”

用^代表bookmark ，应是：

“一二三[标签] ^”

但实际上，由于这里使用的_rangy_库里的_getBookmark_方法比想象中的功能弱了些，只适用于文本节点保存位置，使用该方法得到的结果是：

“一二三^[标签] ”

那么点击3次按钮进行插入的结果就是：

"一二三^ [新标签3][新标签2] [新标签1][标签]"


**一种简单的处理方法**

输入@标签后的**range.commonAncestorContainer**是**TextNode**，而在input标签后的则是editor节点，是**ElementNode**，那么在**完成选人部分**加个判断，是ElementNode都统一定位到末尾进行插入.

以下是一个实用的**移动光标到元素末尾**的方法。

```js
_setCaretToEnd(editor){
	if(editor == null){
		return
	}
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

### 完成选人阶段

#### 在指定的位置插入选中人的标签

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
	range.deleteContents(); // delete origin content in range like '@' or nothing
	// 插入片段前先保存末尾节点供后续重置range范围
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

#### 返回格式化后的文本

```js
extractContents(editor){
	editor  = editor || this.refs.editor.getDOMNode();
	let t = this;
	let nodes = editor.childNodes;
	let content = '';
	if(nodes.length === 0){
		return ''
	}
	// 提取文本、input人物标签和换行
	for (let i = 0, len = nodes.length; i < len; i += 1) {
		let node = nodes[i];
		if (node.nodeType === Node.ELEMENT_NODE) {
			let tagName = node.tagName.toLowerCase();
			if (tagName === 'input') { // input element
				let item = JSON.parse(node.getAttribute('data'));
				content += t.props.formatValue(item);
			} else if (tagName === 'br') {
				content += '\n';
			}else{
				// 递归调用 粘贴过来的html片段可能包含多层节点
				content += t.extractContents(node)
			}
		} else if (node.nodeType === Node.TEXT_NODE) {
			content += node.nodeValue;
		}
	}
	return content
}
``` 

### 自定义bookmark的创建和定位

#### 创建bookmark

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
		// insert end node
		clone = range.cloneRange();
		clone.collapse(false);
		clone.insertNode(endNode);
	}
	// insert start node
	clone = range.cloneRange();
	clone.collapse(true);
	clone.insertNode(startNode);
	// reset range position
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


#### 定位到bookmark

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

#### 进行调用
```js
let selection = rangy.getSelection();
let range = selection.getRangeAt(0);
// 创建bookmark
let bookmark = rangy. createBookmark(range, true);
// ... 完成选人之后需要进行插入标签
// 移动到bookmark位置
rangy.moveToBookmark(bookmark);
```

## 实现过程中的一些问题
- contenteditable元素无法focus

body设置了-webkit-user-select:none导致无法focus，于是在该元素上重设为-webkit-user-select:text

- 在手机（以下均指iPhone6的webview及safari）上点击编辑框focus较慢（需要深度点击）

pc的safari没有该问题，加上了onClick事件来触发focus事件（判断非focus状态才触发）。

另外触发focus会将光标置到开头，这里调用上面提到的setCaretToEnd方法挪到最后就行了。

- enter换行会得到两个br标签，继续输入才会消除第二个br

这里在格式化输入时对br标签进行处理，删除多余的换行

`content.replace(/\n$/, '').replace(/\n\n/g, '\n') `

并且当格式化后的content为空时做一次清空处理，清除遗留的一个br

`editor.innerHTML = ''`

- 在手机上blur事件处理中通过Selection获取range对象阻塞blur事件的后续执行

经过调试发现safari在页面blur后Selection对象的rangeCount为0（chrome中blur后仍保留之前的Selection），所以调用getRangeAt(0)会出现报错。

因此一个解决办法就是在用户输入（如果输入的不是@符号）时就更新bookmark。

## 移动端的使用
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

至于bookmark，我们在上述已经以工具方法的形式进行提供了，那么在定位bookmark时就不用判断commonAncestorContainer了，只要判断标记节点是否存在就可以了。

``` js
if(t._bookmark && document.getElementById(t._bookmark.startNode)){
	range = rangy.moveToBookmark(t._bookmark, range);
}else{
	range = this._setCaretToEnd(editor);
}
``` 

## 总结
初看Range感觉比较冷门，细想一下，在编辑器中还是有比较丰富的应用的，无论是富文本编辑器还是在线代码编辑器，比如[ckeditor](https://github.com/ckeditor/ckeditor-dev/)，都可以基于Range进行实现，不过在浏览器表现上面，即使不考虑IE，有些方面还是存在一些细节和想象中不一样，仍需要进行一定处理，只能且行且注意了。

## 参考资料

>[https://dom.spec.whatwg.org/#introduction-to-dom-ranges](https://dom.spec.whatwg.org/#introduction-to-dom-ranges)

> [https://developer.mozilla.org/en-US/docs/Web/API/range](https://developer.mozilla.org/en-US/docs/Web/API/range)

>[https://github.com/timdown/rangy/wiki/Rangy-Range](https://github.com/timdown/rangy/wiki/Rangy-Range)
