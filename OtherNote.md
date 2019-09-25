
#### 如何准确判断 WebView 加载完成

###### 问题
 正常情况下我们把处理网页加载完毕的代码放在 `- (void)webViewDidFinishLoad:(UIWebView *)webView`里。但 `WebViewDidFinishLoad `时网页真的加载完了吗？

官方文档并没有说明` WebViewDidFinishLoad `到底在什么时候被调用，但事实证明在某些情况下 `WebViewDidFinishLoad` 可能不是你想要的时机。

`WebViewDidFinishLoad` 被调用时，`readyState` 可能处在 `interactive` 和 `complete` 两种状态。当我们需要对网页中的元素进行修改时，最好在 complete 状态进行，不然我们的修改可能被重置。

当网页重定向发生时，网址被重定向几次，`WebViewDidFinishLoad` 就会被调用几次。所以如果你只想在最后加载完成时调用某些代码，可以通过`webView.isLoading`来判断。当` WebViewDidFinishLoad `时如果 `webView.isLoading == YES` 那么说明网页可能发生了重定向。

```
- (void)webViewDidFinishLoad:(UIWebView *)webView {
    if (!webView.isLoading) {
        [self webViewDidFinishLoadCompletely];
    }
}
```


##### web加载的常识

> document.readyState

1. uninitialized : 还没开始加载
2. loading : 加载中
3. loaded : 加载完成
4. *interactive : 结束渲染，用户已经可以与网页进行交互。但内嵌资源还在加载中*
5. *complete : 完全加载完成*


`window.onload `或者 `document.onreadystatechange` 两个方法，他们都可以准确判断内嵌资源加载完毕的时机

#### 方案

```
//以uiwebview举例
- (void)webViewDidFinishLoad:(UIWebView *)webView {
    if (!webView.isLoading) {
        NSString *readyState = [webView stringByEvaluatingJavaScriptFromString:@"document.readyState"];
        BOOL complete = [readyState isEqualToString:@"complete"];
        if (complete) {
            [self webViewDidFinishLoadCompletely];
        } else {
            NSString *jsString =
            @"window.onload = function() {"
            @"    xfNewsContext.onload();"
            @"};"
            @"document.onreadystatechange = function () {"
            @"    if (document.readyState == \"complete\") {"
            @"        xfNewsContext.documentReadyStateComplete();"
            @"    }"
            @"};";
            [_webView stringByEvaluatingJavaScriptFromString:jsString];
        }
        NSLog(@"%@", NSStringFromSelector(_cmd));
    }
}
```

这种方式有缺点就是，他会覆盖js原来的方法。不过这个可以跟web人员沟通一下这个能不能重写。
另外一种方式就是`WKWebView的`属性，`estimatedProgress`

```
This value ranges from 0.0 to 1.0 based on the total number of
 bytes expected to be received, including the main document and all of its
 potential subresources. After a navigation completes, the value remains at 1.0
 until a new navigation starts, at which point it is reset to 0.0.
```
