# 移动端页面调试的方法总结

## 原生的 alert 或把消息文本 write 到某个可见的页面元素中

侵入性强；只能查看文本

## vConsole

https://github.com/WechatFE/vConsole

通过引入 js 文件的方式，在页面上注入调试相关的 UI 来显示劫持到的 console 消息和网络消息

有 js 侵入性和页面侵入性；可以查看文本、object、网络消息

## jsconsole

https://github.com/remy/jsconsole

通过引入 js 文件的方式，在页面上劫持 console 日志，并通过服务端，转发给具有相同 id 的客户端

有 js 侵入性；可以查看文本

## chrome 远程调试

移动端 chrome 的调试信息通过 USB 传给桌面端的 chrome

无侵入性；可以查看非常丰富的调试信息；要求有桌面端浏览器、移动端浏览器、Webview 和 USB 连接；需要翻墙

注意要把 `appspot.com` 加入要翻墙的 URL 规则

在桌面端打开 `chrome://inspect/#devices` 以查看调试信息
