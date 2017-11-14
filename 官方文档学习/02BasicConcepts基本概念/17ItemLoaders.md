## ItemLoader

item加载器提供一个方便的机制把scraped的item填充。尽管items可以使用自己的字典的API，item装载器提供更方便的API用于填充他们抓取的过程中，通过自动化的一些常见任务，如分析原料提取数据之前分配。

换句话说，items提供了数据的容器，而item loaders提供填充该容器的机制。

item loader的目的是提供一种灵活、高效和简单的机制，以扩展或覆盖不同字段解析规则，无论是由蜘蛛还是源格式（HTML、XML等），而不会成为维护的噩梦。

### 使用item加载器去填充items

```python
from scrapy.loader import ItemLoader
from myproject.items import Product

def parse(self, response):
    l = ItemLoader(item=Product(), response=response)
    l.add_xpath('name', '//div[@class="product_name"]')
    l.add_xpath('name', '//div[@class="product_title"]')
    l.add_xpath('price', '//p[@id="price"]')
    l.add_css('stock', 'p#stock]')
    l.add_value('last_updated', 'today') # you can also use literal values
    return l.load_item()
```
这个创建一个ItemLoader加载器，指定了Product这个item类。就是加载器要填充的items，然后使用add_xpath,或者add_css去填充加载器，最后调用load_item 去得到item. 然后返回一个Item对象。这里就是一个Porduct对象。

上面我们可以看出来，name字段使用了2个add_xpath,追加结果。下一个部分有详细描述。

```python
    def _add_value(self, field_name, value):
        value = arg_to_iter(value)
        processed_value = self._process_input_value(field_name, value)
        if processed_value:
            self._values[field_name] += arg_to_iter(processed_value)
    def add_xpath(self, field_name, xpath, *processors, **kw):
        values = self._get_xpathvalues(xpath, **kw)
        self.add_value(field_name, values, *processors, **kw)
        
    def load_item(self):
        item = self.item
        for field_name in tuple(self._values):
            value = self.get_output_value(field_name)
            if value is not None:
                item[field_name] = value

        return item
````
### 输入输出处理器
```
l = ItemLoader(Product(), some_selector)
l.add_xpath('name', xpath1) # (1)
l.add_xpath('name', xpath2) # (2)
l.add_css('name', css) # (3)
l.add_value('name', 'test') # (4)
return l.load_item() # (5)
```
1. : 数据从xpath1被提取出来，通过输入处理器name字段，这个结果保存到itemloader中。 但是还没有保存到item中。

1. ：数据从xpath提取出来。通过和（1）相同的输入处理器，，然后结果被追加到itemloader中。
1. ： 同上，继续追加
1. ： 这个情况和上面的基本一样，只是这个是直接赋值的，不用xpath提取了。这个赋值的不一定是可迭代的，但是输入处理器必须是收到可迭代的。
1. ：在上面的4个步骤中， 数据都收集到itemloader中， 然后通过输出处理器，设置一个item。

值得注意的是，处理器只是可调用对象，它们被调用要解析的数据，并返回解析的值。因此，您可以使用任何函数作为输入或输出处理器。唯一的要求是，它们必须接受一个（只有一个）位置参数，这将是一个迭代器。 

### 定义item加载器
```python
from scrapy.loader import ItemLoader
from scrapy.loader.processors import TakeFirst, MapCompose, Join

class ProductLoader(ItemLoader):

    default_output_processor = TakeFirst()

    name_in = MapCompose(unicode.title)
    name_out = Join()

    price_in = MapCompose(unicode.strip)

    # ...
```
这里我们想给我们的一个item类product创建一个加载器， 定义一个加载器类。继承了ItemLoader，
* 设置了默认的输出处理器为TakeFirst()方法，
* name字段的输入处理器为MapCompose(unicode.title)
* name字段的输出处理器为 Join()
* price字段的输入为MapCompose(unicode.strip)

这几个方法到底都是啥呢
```python
class Join(object):

    def __init__(self, separator=u' '):
        self.separator = separator

    def __call__(self, values):
        return self.separator.join(values)
class TakeFirst(object):

    def __call__(self, values):
        for value in values:
            if value is not None and value != '':
                return value

class MapCompose(object):

    def __init__(self, *functions, **default_loader_context):
        self.functions = functions
        self.default_loader_context = default_loader_context

    def __call__(self, value, loader_context=None):
        values = arg_to_iter(value)
        if loader_context:
            context = MergeDict(loader_context, self.default_loader_context)
        else:
            context = self.default_loader_context
        wrapped_funcs = [wrap_loader_context(f, context) for f in self.functions]
        for func in wrapped_funcs:
            next_values = []
            for v in values:
                next_values += arg_to_iter(func(v))
            values = next_values
        return values     
```
这几个方法的代码都很简单
* join函数内部调用了str的函数。把一个数组元素通过指定的连接符连接起来。["a","b","c"] => "a b c"
* TakeFirst这个方法，就是找到数组中的第一个非none的元素就返回[None,"a","b"] => "a"
* MapCompose根据一个函数一个函数的处理，将结果一个一个合并起来。最终返回。

上面我们的加载器类设置了默认的输入和输出器。Itemloader的默认是
```python
class ItemLoader(object):
    #省略
    default_input_processor = Identity()
    default_output_processor = Identity()
    ##省略
```
看看Identity是啥
```python
class Identity(object):

    def __call__(self, values):
        return values
```
这个方法，就是传递啥返回啥。

### 定义一个输入输出器

```python
import scrapy
from scrapy.loader.processors import Join, MapCompose, TakeFirst
from w3lib.html import remove_tags

def filter_price(value):
    if value.isdigit():
        return value

class Product(scrapy.Item):
    name = scrapy.Field(
        input_processor=MapCompose(remove_tags),
        output_processor=Join(),
    )
    price = scrapy.Field(
        input_processor=MapCompose(remove_tags, filter_price),
        output_processor=TakeFirst(),
    )
```
```python
>>> from scrapy.loader import ItemLoader
>>> il = ItemLoader(item=Product())
>>> il.add_value('name', [u'Welcome to my', u'<strong>website</strong>'])
>>> il.add_value('price', [u'&euro;', u'<span>1000</span>'])
>>> il.load_item()
{'name': u'Welcome to my website', 'price': u'1000'}
```
name字段的获取过程： 在输入处理器用了remove_tags函数，把<strong></strong>这些符号去去掉了。在输出处理器上使用join合并了2个str.
price字段的获取过程： 在输入处理器用remove_tags函数，把tags去掉，在是哦那个filter_price函数，那输入处理器给loader的值是None,1000， 然后在输出处理器使用TakeFirst，就得到了1000。


### item加载器上下文

```python
    loader = ItemLoader(product)
    loader.context['unit'] = 'cm'
```

```python
    loader = ItemLoader(product, unit='cm')
```

```python
    class ProductLoader(ItemLoader):
        length_out = MapCompose(parse_length, unit='cm')
```

### Itemload类
#### get_value(value, *processors, **kwargs)
将value进程processors方法一个一个处理
```python
>>> from scrapy.loader.processors import TakeFirst
>>> loader.get_value(u'name: foo', TakeFirst(), unicode.upper, re='name: (.+)')
'FOO`
```
经过3个processors处理流程。
u'name: foo' ===> u'foo' ====> u'foo'===> FOO

源码如下
```python
    def get_value(self, value, *processors, **kw):
        regex = kw.get('re', None)
        if regex:
            value = arg_to_iter(value)
            value = flatten(extract_regex(regex, x) for x in value)

        for proc in processors:
            if value is None:
                break
            proc = wrap_loader_context(proc, self.context)
            value = proc(value)
        return value
```
这个代码是先处理正则的，然后在处理processor，遍历处理。最后返回。不管他写在前面还是后面都是第一个处理。

#### add_value(field_name, value, *processors, **kwargs)

源码如下
```python
    def add_value(self, field_name, value, *processors, **kw):
        value = self.get_value(value, *processors, **kw)
        if value is None:
            return
        if not field_name:
            for k, v in six.iteritems(value):
                self._add_value(k, v)
        else:
            self._add_value(field_name, value)
    def _add_value(self, field_name, value):
        value = arg_to_iter(value)
        processed_value = self._process_input_value(field_name, value)
        if processed_value:
            self._values[field_name] += arg_to_iter(processed_value)
```
简单看下源码，然后我们分析一个样例把。
```
loader.add_value('name', u'name: foo', TakeFirst(), re='name: (.+)')
```
这个，在调用add_value的时候， 先调用get_value函数，得到foo
如果没之前没有设置过name字段， 就添加一个，如果有，调用_add_value.合并到在loader的对应name字段中列表中。
### replace_value(field_name, value, *processors, **kwargs)
这个很好立即的。 上面我们使用add是添加到列表中， 这个是替换了。
```python
 def replace_value(self, field_name, value, *processors, **kw):
        value = self.get_value(value, *processors, **kw)
        if value is None:
            return
        if not field_name:
            for k, v in six.iteritems(value):
                self._replace_value(k, v)
        else:
            self._replace_value(field_name, value)
def _replace_value(self, field_name, value):
    self._values.pop(field_name, None)
    self._add_value(field_name, value)
```
如果原来使用过add_value,使用pop原来的。然后使用add_value去添加。

#### get_xpath(xpath, *processors, **kwargs)
这个和get_value类似。 只是这个地方接受的是一个xpath表达式。不是现在的value值。
#### add_xpath(field_name, xpath, *processors, **kwargs)
这个和add_value类似
#### replace_xpath(field_name, xpath, *processors, **kwargs)
这个是和replace_value类似

**同样也有css的一套方法，这里就不写了，太麻烦了**