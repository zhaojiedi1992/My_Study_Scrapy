## DownloadMiddleware下载中间件

### 内置的下载中间件参考

#### CookiesMiddleware
##### class scrapy.downloadermiddlewares.cookies.CookiesMiddleware
此中间件允许处理需要cookie的站点，例如使用会话的站点。它跟踪Web服务器发送的cookie，然后像Web浏览器那样在后续请求（从那个蜘蛛）发送回它们。

可用的设置
* COOKIES_ENABLED :默认True,如果禁用，就不会给服务器发送cookie信息
* COOKIES_DEBUG：默认false,如果启用，scrapy会记录所有cookies信息发送。

##### 一个爬虫多个cookie会话
这是支持一个爬虫多个cookie会话的，使用cookiejar在request的meta中，默认使用的单个cookie jar，你可以指定启用cooked会话

```python
for i, url in enumerate(urls):
    yield scrapy.Request(url, meta={'cookiejar': i},
        callback=self.parse_page)
```

这样使用cookiejar不是粘性的， 你如果在parse方法使用Request方法，必须继续指定cookiejar
```python
def parse_page(self, response):
    # do some processing
    return scrapy.Request("http://www.example.com/otherpage",
        meta={'cookiejar': response.meta['cookiejar']},
        callback=self.parse_other_page)
```

启用COOKIES_DEBUG会得到如下信息

```cmd
2011-04-06 14:35:10-0300 [scrapy.core.engine] INFO: Spider opened
2011-04-06 14:35:10-0300 [scrapy.downloadermiddlewares.cookies] DEBUG: Sending cookies to: <GET http://www.diningcity.com/netherlands/index.html>
        Cookie: clientlanguage_nl=en_EN
2011-04-06 14:35:14-0300 [scrapy.downloadermiddlewares.cookies] DEBUG: Received cookies from: <200 http://www.diningcity.com/netherlands/index.html>
        Set-Cookie: JSESSIONID=B~FA4DC0C496C8762AE4F1A620EAB34F38; Path=/
        Set-Cookie: ip_isocode=US
        Set-Cookie: clientlanguage_nl=en_EN; Expires=Thu, 07-Apr-2011 21:21:34 GMT; Path=/
2011-04-06 14:49:50-0300 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://www.diningcity.com/netherlands/index.html> (referer: None)
[...]
```