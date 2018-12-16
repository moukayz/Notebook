# lxml库

* lxml可直接读取html文件，不需要其他操作
* lxml需要自己的编译器来解析html文件

```python
parser = etree.HTMLParser(encoding="gbk")
html = etree.parse(html, parser=parser)  # 输入为html文件，增加解析器
```

* lxml返回的结果需要利用`tostring`将单个对象转换为字节（不能直接对list使用此方法），再利用`decode`解码为文字。

```python
answers = html.xpath('//*/div[@class="l-tbody"]/div[@class="art-text"]/p')
answer = ""
for i in answers:
    i = etree.tostring(i, encoding='gbk')
    i = i.decode('gbk')
    i = re.sub(r' |\n|\r|\t|&#13|;|', "", i)
    answer = answer + "\n" + re.sub(r'<.*?>', front, i)
```

