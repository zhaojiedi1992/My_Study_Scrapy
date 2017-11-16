## 请求和响应
Request和Response这2个类是抓取web的基础类。

### Request objects

#### class scrapy.http.Request(url[, callback, method='GET', headers, body, cookies, meta, encoding='utf-8', priority=0, dont_filter=False, errback, flags])
这个类用于构造一个http请求，通常在爬虫里面生成的，被下载器执行，最终生成一个Response。

详细参数
* url: 请求的url
* callback:响应流的回调方法
* method: 请求方法，默认是'GET'
* meta:用于和响应流交互数据
* body:请求body信息
* header:请求头信息
* cookies:cookies信息
    ```python
    request_with_cookies = Request(url="http://www.example.com",
                               cookies={'currency': 'USD', 'country': 'UY'})
    request_with_cookies = Request(url="http://www.example.com",
                               cookies=[{'name': 'currency',
                                        'value': 'USD',
                                        'domain': 'example.com',
                                        'path': '/currency'}])
    ```
* encoding:默认是utf-8的
* priority:请求的优先级，影响调度，正数提高优先级，负数降低。
* dont_fileter: 不过滤，默认是False:也就是会过滤掉重复的
* errback:错误的回调
* flags:发送到请求的标志，可用于日志或其他用途。

