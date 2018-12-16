# os库

## os.path模块

* `os.walk`  遍历一个文件夹下的全部文件和文件夹（包括子文件）

```python
for root, dirs, files in os.walk(dirname, topdown=True):
```

* `os.listdir(dirname)`找到一个文件内的全部文件夹和文件，不包括子文件夹中的内容，返回值为list
* `os.path.isdir(dirname)`判断是文件夹还是文件

```python
dirname = 'C:\\Users\\daiyifan\\Desktop\\daiyifan\\pcbaby\\data'
```

* `os.path.exits(dirname)`判断某路径是否存在
* `os.makedirs(dirname)`创造文件夹，包括文件夹内的子文件夹         
* `os.makedir(dirnam)`创造文件夹，但上一层级的文件夹不存在会报错 
* 将html文件写入本地时候，应判断该文件夹是否存在（参见pcbaby）

```python
def download_html(path,response):
    print(path)
    filename = path.split("/")[-1]
    print(filename)
    path = path.replace('https://', "")
    path = re.sub(r'/\d*\.html', '', path)
    path = "data/" + path
    if os.path.exists(path):
        print("该路径已存在")
    else:
        os.makedirs(path)
    with open(path + "/" + filename, "wb") as f:
        f.write(response.body)
```



