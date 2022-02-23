---
picture: image/tabs_thum.png
tags: [妙招, 网页]
date: 2022-02-23
---
# [生活小妙招] 禁止网站打开新的Tab页

当我们浏览网页一段时间后，你一定见过下面的场景, 浏览器会产生大量的Tab页，当我们会后再找自己的历史内容时，往往会迷失其中。

![tabs](image/禁止网站打开新的Tab页/tabs.jpg "tab")

> 这个问题的主要原因是，很多开发者滥用网页链接的_blank 特性，打开了很多没有意义的Tab页面。很多网站开发人员为了兼容性，没能正确衡量连接的_blank场景。我们可以通过以下脚本强制网页连载在当前页面打开。

## 解决方法

安装自定义脚本插件，让网页默认在当前页面打开链接，如果需要在新tab页打开，则鼠标右键手动在新的Tab页打开即可。

步骤：

1. 安装 Userscripts 插件
2. 复制脚本到 Userscripts

### 1. 安装 Userscripts 插件

以下是 chrome 和 Safari插件地址：

Safari 安装插件：https://apps.apple.com/us/app/userscripts/id1463298887

Chrome 安装插件：https://chrome.google.com/webstore/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo

### 2. 复制脚本到插件

安装好插件下，可以点击新建脚本，复制下面的内容到脚本中，点击保存。 然后刷新页面，页面中的所有链接就可以只在当前页面打开了

```js
// ==UserScript==
// @name        NoTargetBlank
// @description All links open in current tab，diable web open new tab
// @version      0.1
// @author       github.com/ti/
// @match       *://*/*
// ==/UserScript==
function removeAllTargetBlank(delay, maxCount, count) {
    if (count && count >= maxCount) {
        return
    }
    if (!count) {
        count = 1;
    }
    document.querySelectorAll('a[target="_blank"]').forEach((v) => {
        v.removeAttribute('target');
    });
    setTimeout((d, m, c) => {
        d = 1000;
        c++;
        removeAllTargetBlank(d, m, c);
    }, delay, delay, maxCount, count);
}
removeAllTargetBlank(200, 10);
```

> 备注：脚本的主要逻辑是，在页面载入200毫秒后，查找页面中的所有target为blank的链接，移除其target属性。再之后每隔一秒执行一次这样的行为，10秒后停止。之所以这样做是因为很多页面内容是异步加载的。页面完整载入耗时较长。采用延时模式是为了尽可能不影响页面的加载速度。
