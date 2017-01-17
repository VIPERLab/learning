# nginx核心模块官方文档
[ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html)

[ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)


# 春哥对if指令的详细解释
[how-nginx-location-if-works](http://agentzh.blogspot.com/2011/03/how-nginx-location-if-works.html)


# 春哥的nginx教程
[agentzh-nginx-tutorials-zhcn](https://openresty.org/download/agentzh-nginx-tutorials-zhcn.html)

- [Nginx 处理请求的过程一共划分为 11 个阶段](https://openresty.org/download/agentzh-nginx-tutorials-zhcn.html#02-NginxDirectiveExecOrder08)，按照执行顺序依次是 post-read、server-rewrite、findconfig、rewrite、post-rewrite、preaccess、access、post-access、try-files、content 以及 log.

|阶段|说明|支持注册模块|
|---:|:---|---|
|post-read|在 Nginx 读取并解析完请求头（request headers）之后就立即开始运行. 典型的有ngx_realip.| 支持|
|server-rewrite|执行server配置块中的rewrite/set指令|支持|
|find-config|完成当成请求和location配置之间的配对|不支持|
|rewrite|location配置块中的rewrite/set指令|支持|
|post-rewrite|如果需要, 完成rewrite阶段所要求的内部跳转|不支持|
|preaccess|频度, 并发度控制. 典型的有ngx_limit_req, ngx_limit_zone, ngx_realip|支持|
|access|执行访问控制性质的任务, 如鉴权, 验IP. 一定要搭配content阶段指令, 否则ngx_static模块有可能会起作用.|支持|
|post-access|配合 access 阶段实现标准 ngx_http_core 模块提供的配置指令 satisfy 的功能.|不支持|
|try-files|专门用于实现标准配置指令 try_files 的功能.|不支持|
|content|内容处理程序, 每个location只能有一个内容处理程序|支持|

- [ngx_realip模块的作用描述](https://openresty.org/download/agentzh-nginx-tutorials-zhcn.html#02-NginxDirectiveExecOrder08): 当 Nginx 处理的请求经过了某个 HTTP 代理服务器的转发时，这个模块就变得特别有用。当原始的用户请求经过转发之后，Nginx 接收到的请求的来源地址无一例外地变成了该代理服务器的 IP 地址，于是 Nginx 以及 Nginx 背后的应用就无法知道原始请求的真实来源。所以，一般我们会在 Nginx 之前的代理服务器中把请求的原始来源地址编码进某个特殊的 HTTP 请求头中（例如上例中的 X-My-IP 请求头），然后再在 Nginx 一侧把这个请求头中编码的地址恢复出来。这样 Nginx 中的后续处理阶段（包括 Nginx 背后的各种后端应用）就会认为这些请求直接来自那些原始的地址，代理服务器就仿佛不存在一样。
> 后续可以考虑使用ngx_realip模块把用户IP透到逻辑层)
