# Scrapy-Xpath

###  标签\[1\]与extract\_first\(\)

标签\[1\]的覆盖范围更广泛,比如

```python
//div[@class=title]/a[1]   #所有div标签中的第一个a
//div[@class=title]/a.extract_first()  #所有的div标签下的a标签构成的列表的第一个元素
```

###  as=response.xpath\(//\*/div\[@class="guide"\]\)

 xpath返回的迭代器有三个部分

```python
[<Selector xpath='descendant-or-self::title' data='<title>Quotes to Scrape</title>'>]
```

* `extract()`函数则是把其中的data部分全部提取出来，是一个列表；
* `extract_first()`返回符合条件的第一个数据；
* `as=response.xpath('//*/div[@class="guide"]/text()').extract()`加入text后，可以去掉元素的标签
* 全部孩子节点“//”；
* 直接孩子节点“/”
* '\*' 返回全部符合条件的节点进入列表。

###  利用for循环，对上述符合条件的节点进行筛选

```python
# 在使用extract之前才可以继续筛选
as=response.xpath('//*/div[@class="guide"]')
for a in as：
     a.xpath("p/text()")
```

###  re模块的替换

 `re.sub(正则表达式，替换成的部分，字符串)`；

sub是贪心算法，会匹配到最长的模式，在正则表达式中加入”？“则使用最短匹配方式；寻找最近的&lt;&gt;之间来替换

 `a = re.sub(r'<.*?>', '', a)`

###  xpath选择某标签的属性值

```python
answer_xpath.xpath("center/img/@src").extract()
```

###  打开json文件必须在同一目录

```python
with open("C:\\Users\\daiyifan\\pclady_wiki\\pclady_wiki\\wiki_url.json","r",encoding="utf-8") as js:
    wiki_url=json.load(js)
```

###  选取属于 bookstore 子元素的最后一个 book 元素

 `/bookstore/book[last()]`

###  选取带有属性的div节点

 `/div[@*]`

### 分层次过滤节点

```python
as=response.xpath("//*/div/p")
for a in as:
     i=a.xpath("span").extract()
     j=a.xpath("@href").extract()
```

###  元素的兄弟节点

 `following-sibling::p #后兄弟节点P，属于xpath的轴`

