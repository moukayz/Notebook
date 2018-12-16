# Scrapy安装与开始项目

打开命令行

进入需要创建虚拟环境的所属文件夹

创建虚拟环境：

```bash
python -m venv 虚拟环境名称
```

进入虚拟环境

```bash
虚拟环境名\scripts\activate
```

在虚拟环境中安装包（scrapy）

```bash
pip install scrapy
```

利用包开展项目

```bash
scrapy startproject tutorial
```

 **备注：scrapy需要c++ 14.0与pywin32（包）**

在命令行中运行scrapy的项目，应进入该项目的虚拟环境，再进入第一个toturial（即与idea/venv并行的项目文件）后写`scrapy crawl quotes`（爬虫的名字），在命令行中选择文件元素使用

`scrapy shell 'http://quotes.toscrape.com/page/1'` 

 在css中，使用`extract()`和`exteact_first()`方法来得到html文件的元素，或者使用`re()`方法根据正则表达式提取html文件元素

 可以在spriders中写extract等需要的元素，在命令行中执行, 运行爬虫，并存入名为quotes的json文件

```bash
scrapy crawl quotes -o quotes.json
```

找到sprider正在爬的网页使用response.url，在sprider中，url列表名称不能变必须为start\_urls

 中止正在运行的sprider，ctrl+c，继续该sprider （somespider指sprider名字**（PS：此种回复不能续写json）**）：

```bash
scrapy crawl somespider -s JOBDIR=crawls/somespider-1
```

 可续写json的命令，json为原始json名

```bash
scrapy crawl answers -s JOBDIR=crawls/answers-1 -o answers.json
```

 爬取的数据写入csv后无法显示中文的问题，可通过**导入mongodb数据库再导出**解决

 可以把网页整个保存成为html文件，方便后期处理：在spider中这样写

```python
def parse(self, response):
    filename = re.sub(r'https://www.pclady.com.cn/wiki/',"",response.url)
    with open(filename, 'wb') as f:
        f.write(response.body)
```

 scrapy的可以将当前url放入spider类的其他方法，其中dont\_filter指的可以重复爬取网页。

```python
response.follow(url, callback=self.final_parse,dont_filter=True)
```

