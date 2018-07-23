webview启动过程
> * 无反馈：webview初始化
>   * 全局webview
> * 白屏：网络建立，dns，下载
>   * 客户端代理请求数据
>   * 域名统一，减少dns解析时间
>   * Transfer-Encoding: chunked
> * loading: 解析脚本，接收数据，渲染
>   * 框架初始化时间在移动端不可忽视(如react)
>   * code split
>   * 关键css
>   * ssr
> * 展现

计算首屏时间/白屏时间
> * [performance.timing](https://www.w3.org/TR/navigation-timing/)

Document event
> * DOMContentLoaded HTML文档被完全加载和解析完成之后
> * load 页面完全加载完成
> * document.readyState
>   * uninitiallized 未载入
>   * loading 载入中
>   * loaded 部分文件已加载,对象模型未生效
>   * interactive 已载入,可以开始交互
>   * complete 载入完成
> * 顺序(?存疑)
>   * interactive 》DOMContentLoaded 》 complete 》 onload