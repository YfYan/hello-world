# 知乎爬虫

一个知乎用户可能自己关注的话题不多，为了获得更多的信息，可以爬取该用户关注者有关的问题、专栏等内容，进而可对该用户做用户画像。

代码在github上只给了最终版本version 2.0的，那个最快了。

## 知乎url的基本分析

知乎的url是非常有规律的

用户的url为： https://www.zhihu.com/people/ + 用户token

问题的url为：https://www.zhihu.com/question/ + 问题id

专栏的url为：https://zhuanlan.zhihu.com/ + 专栏id

因此，爬取的一个关键是获取这些token和id，之后就是通过获得的url爬取信息。

# version 1.0

##用户的关注者获取

![pic1](http://github.com/YfYan/YfYan.github.io/raw/master/images/pic1.png)

作为一个只初步了解爬虫的新手，一开始的基本思路自然是requests+BeautifulSoup。观察用户的关注者页面，翻页时是在url后加 ?page=X （X为页面），每一页有20个关注者，X的最大值根据关注者数量获得。然而真正跑的时候，BeauifulSoup通过正则表达只能在每个页面只能找出三个关注者，因为知乎的页面大多数都是异步加载的，右击页面的源代码里信息的很少的，页面的大多数信息都是JavaScript异步渲染的。念及我两天速成的js语法，我头大了。幸好找了半天，配置好了我实习阶段的第一个神器——selenium。 



selenium是一个web测试库，可以带动一个真实的Chrome，为网页中的JavaScript提供运行环境。这样以来，通过一个webdriver驱动一个Chome加载页面，这样之后通过page_source得到的页面html就是渲染过的了。



尽管如何，selenium使用过程也有很多坑。安装电脑Chrome版本对应的驱动到/usr/bin（mac用户）目录下，查看Chrome的版本：在空页面地址栏输入chrome://version，之后下载相应的版本即可。之后用用selenium打开知乎的url，知乎还是会报环境异常相关的错误，之后在运行前添加如下代码可正常运行：

```python
from selenium import webdriver
from selenium.webdriver import ChromeOptions
option = ChromeOptions()
option.add_experimental_option('excludeSwitches', ['enable-automation'])
driver = webdriver.Chrome(options=option)
```

另外为提高速度，加载完一个页面，不要将webdriver关闭，因为每次打开Chrome都需要花费额外的时间，不如等所有的页面都搞定了再关掉webdriver。webdriver本身是可以通过DOM来定位xpath，但似乎这样会耗费更多的资源，所以本人还是用BeautifulSoup做正则表达。

## 关注者的动态获取

知乎用户的动态是可以通过鼠标往下拖再加载的，显然也是动态加载的，这里selenium可以模拟鼠标滚轮往下滑，滑一次可以多下载7个内容，同时要暂停一小段时间，否则会来不及加载。另外，图片信息是没有用的，不妨禁用图片加载，实现如下：

```python
import time
from selenium import webdriver
from selenium.webdriver import ChromeOptions
option = ChromeOptions()
prefs = {"profile.managed_default_content_settings.images":2} #禁止加载图片
option.add_experimental_option("prefs",prefs)
option.add_experimental_option('excludeSwitches', ['enable-automation'])
driver = webdriver.Chrome(options=option)
for count in range(scroll_time): #scroll_time是预设的滚动次数
    driver.execute_script("window.scrollBy(0,3000)") #模拟鼠标滚轮滚动
    time.sleep(sleep_time) #暂停一小段时间供加载 实验得sleep_time最小0.2秒
```

这样就可以获得用户的问题、专栏等的url。

## 问题信息的获取





