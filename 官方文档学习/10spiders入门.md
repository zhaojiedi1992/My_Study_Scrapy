# Spider介绍
## Spider简介
Spider是一个爬虫基类， 所有的爬虫类都直接或者间接继承这个类。
## scrapy.Spider学习
这个类是爬虫的基类，其他的爬虫都必须间接或者直接继承这个类，这个类基本不需要提供具体的方法实现。
下面对Spider主要的属性和方法介绍下，可能会带一些源码分析。
### `name`
这个就是设置爬虫名字的，这个值必须是工程级别唯一的，这个字段也是必须设置的。在Python2中，name必须是ascii值。
### `name 源码分析`
在`scrapy.spiders.__init__.py`文件中有如下代码：
```python
    name = None
    custom_settings = None

    def __init__(self, name=None, **kwargs):
        if name is not None:
            self.name = name
        elif not getattr(self, 'name', None):
            raise ValueError("%s must have a name" % type(self).__name__)
        self.__dict__.update(kwargs)
        if not hasattr(self, 'start_urls'):
            self.start_urls = []
```
我们可以看到初始化方法，设置name属性，然后判断属性是否是None，如果是那就抛出一个异常。
### allowed_domain 
这个是可选的， 如果设置了，请求的url必须是这个域内的地址，超出这个域的过滤掉。
### start_urls
这个不是必须的，但是我们写爬虫就是要爬取数据，不写启动的url，那不搞笑吗，这个就是设置我们爬虫启动的url列表，list类型。
### custom_settings
这个是自定义设置的， 我们有默认的工程设置， 但是工程内有多个爬虫， 我们想给部分爬虫设置单独的设置，就需要这个属性的设置了。
### crawler
该属性由初始化类之后由from_crawler（）类方法设置，并链接到此蜘蛛实例绑定到的Crawler对象。
Crawlers在项目中封装了大量组件，用于单一访问（例如扩展，中间件，信号管理器等）。
### settings
获取爬虫的设置信息
### logger
根据爬虫名字创建的一个log对象。
### from_crawler(crawler, *args, **kwargs)
这是一个类方法，去创建你的爬虫。
### start_requests()
默认实现是遍历start_urls的url，执行Request(url, dont_filter=True)，我们可以不指定start_urls，在这里设置我们的url列表并遍历url列表。
### parse(response)
这是默认的下载响应流的回调方法
返回值只能是Request,dict,Item对象。
### log(message[, level, component])
这个就是写日志的。
### closed(reason)
关闭爬虫，发送关闭信号。
## 使用样例
```python
import scrapy


class MySpider(scrapy.Spider):
    name = 'example.com'
    allowed_domains = ['example.com']
    start_urls = [
        'http://www.example.com/1.html',
        'http://www.example.com/2.html',
        'http://www.example.com/3.html',
    ]

    def parse(self, response):
        self.logger.info('A response from %s just arrived!', response.url)
```
这是最简单的样例了。
```python
import scrapy
from myproject.items import MyItem

class MySpider(scrapy.Spider):
    name = 'example.com'
    allowed_domains = ['example.com']

    def start_requests(self):
        yield scrapy.Request('http://www.example.com/1.html', self.parse)
        yield scrapy.Request('http://www.example.com/2.html', self.parse)
        yield scrapy.Request('http://www.example.com/3.html', self.parse)

    def parse(self, response):
        for h3 in response.xpath('//h3').extract():
            yield MyItem(title=h3)

        for url in response.xpath('//a/@href').extract():
            yield scrapy.Request(url, callback=self.parse)
```
不使用start_urls属性，我们自己重写start_request方法，自己去遍历urls列表。
## 爬虫参数
设置爬虫参数，我们只需要-a选项即可。
使用如下
```cmd
scrapy crawl myspider -a category=electronics
```
然后在我们的爬虫的__init__方法中即可使用
```python
import scrapy

class MySpider(scrapy.Spider):
    name = 'myspider'

    def __init__(self, category=None, *args, **kwargs):
        super(MySpider, self).__init__(*args, **kwargs)
        self.start_urls = ['http://www.example.com/categories/%s' % category]
        # ...
```
我们使用命令行传递过来的参数，默认会直接设置self.category=category ,我们不需要再设置的。这个在我们的start_urls不确定，需要传递的参数才能确认的时候才用的。

## 源码分析
