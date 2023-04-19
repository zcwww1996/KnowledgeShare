[TOC]

# IP代理的构建


```python
import urllib.request
ip="54.39.24.37:3128"
proxy=urllib.request.ProxyHandler({"http":ip})
opener=urllib.request.build_opener(proxy,urllib.request.HTTPHandler)
urllib.request.install_opener(opener)
url="http://www.baidu.com"
data=urllib.request.urlopen(url).read().decode("utf-8","ignore")
print(len(data))
#注意下面
#出现UnicodeEncodeError: 'gbk' codec can't encode character '\xbb' in position 29542: illegal multibyte sequence
#在下面这条指令加上encoding="utf-8"
fh=open("C:\\Users\\gaoxingyuan\\Desktop\\baidu.html","w",encoding="utf-8")
fh.write(data)
fh.close()
```

# ip代理池构建
## 第一种方案(适合于代理IP稳定的情况)

```python
import random
import urllib.request

ippools=[
    "54.39.24.37:3128",
    "36.255.189.38:8080",
    "191.102.73.38:8080",
    ]
def ip(ippools):
    thisip=random.choice(ippools)
    print(thisip)
    proxy=urllib.request.ProxyHandler({"http":thisip})
    opener=urllib.request.build_opener(proxy,urllib.request.HTTPHandler)
    urllib.request.install_opener(opener)
for i in range(0,3):
    try:
        ip(ippools)
        url="http://www.baidu.com"
        data=urllib.request.urlopen(url).read().decode("utf-8","ignore")
        print(len(data))
        fh=open("C:\\Users\\gaoxingyuan\\Desktop\\baidu"+str(i+1)+".html","w",encoding="utf-8")
        fh.write(data)
        fh.close()
    except Exception as e:
        print(str(e))
```


## 第二种方案（接口调用法，这种方法更适合代理IP不稳定的情况）

```python
import urllib.request

def ip():
    thisip=urllib.request.urlopen("API接口地址")
    print(thisip)
    proxy=urllib.request.ProxyHandler({"http":thisip})
    opener=urllib.request.build_opener(proxy,urllib.request.HTTPHandler)
    urllib.request.install_opener(opener)

for i in range(0,3):
    try:
        ip()
        url="http://www.baidu.com"
        data=urllib.request.urlopen(url).read().decode("utf-8","ignore")
        print(len(data))
        fh=open("C:\\Users\\gaoxingyuan\\Desktop\\baidu"+str(i+1)+".html","w",encoding="utf-8")
        fh.write(data)
        fh.close()
    except Exception as e:
        print(str(e))
'''
```
遍历列表
for item in 列表名
    print(item)
'''