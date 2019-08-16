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

## 用户的关注者获取

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

## 关注者动态的获取

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

![pic4](http://github.com/YfYan/YfYan.github.io/raw/master/images/pic4.png)

上图是一个问题的图片，问题信息分为三部分，在上图中从上到下分为话题标签、标题、内容。很不幸，问题页面也是动态加载的，在Chrome中检查这三个部分，均不能在源代码中找到。如果这个也用selenium去做加载页面，实在是太慢了。观察页面的源代码，话题标签和标题在一个meta中可以找到，而较为完整的内容可以在源代码中的JavaScript中找到，用正则表达式可找。

![pic5](/Users/xijinping/Downloads/gitwork/YfYan.github.io/images/pic5.png)

![pic6](/Users/xijinping/Downloads/gitwork/YfYan.github.io/images/pic6.png)

爬取问题内容的正则表达式如下，目前使用还可以，但以后万一js改了需要进一步维护

```python
import re
pattern = re.compile(',"excerpt":"([\s\S]*)","commentPermission"') 
content=pattern.findall(script.text)[0] #script.text是页面源代码
```



# version 1.1

一个最原始的爬虫已经可以跑了，但问题是实在是太慢了，1.1版本主要是增加了多线程的功能。在并行处理方面，python主要有三种不同粒度的并行方法：多进程（muti-processing）、多线程（muti-threading）、协程（asyncio）。



多进程是进程数要和CPU核数相同，进程间通信较为麻烦。python中自带了封装好的mutiproccssing包，但mutiproccssing要在top of the module，否则会出can‘t pickel的问题，幸好老板给了[原因和解决方案](https://stackoverflow.com/questions/925100/python-queue-multiprocessing-queue-how-they-behave/925241#925241)，不然实在是一头雾水。



多线程是我最终采用的多线程方案，之后的各种任务我能多线程就多线程，看到spyder右下角CPU使用率90+非常爽。多线程的坑在于是否线程安全，python自带的数据结构，比如列表、字典都是线程安全的，但自己写的一些东西都不一定了，这个之后还会提到。个人觉得多线程的使用还是很灵活的，常用套路如下：

```python
import threading
class My_threading(threading.Thread):
  def __init__(self,arg):
    threading.Thread.__init__(self)
    self.args=args #arg是自定义函数的参数,事先先分割好
  def run(self):#重载run()方法
    my_function(self.arg) #my_function是想要多线程的函数
ths = [My_threading(args[i]) for i in range(thread_num)] #thread_num是线程数
for th in ths:
  th.start()
for th in ths:
  th.join()
```



协程据老板说也是非常有用的一个方法，可惜本人用的spyder对acyncio的支持不是很好，跑的时候会报错“RuntimeError: This event loop is already running”，在stackoverflow上查到一个方法说是可以解决，但我这儿似乎还是不work，这个坑暂时就先放着了，反正多线程挺好用的。

```python
import nest_asyncio
nest_asyncio.apply()
import asyncio
```



# version 1.2

多线程后速度大大提高，但又出了新的问题，爬的多了会出现验证码，实验后发现，只要在出现验证码后输入，再重新get那个url，就可以继续爬取了。多线程的时候，只要有一个线程的验证码输入完毕，其他线程的验证码问题也会自动解决。知乎登陆时的验证码是点击倒立的字，幸好这里没有这么麻烦，是输入四个字符。顺便一提，验证码图片的格式是base64的，非常有趣的一种用文本表示图片的方法，可以用base64库去解决。

<center>
<figure class="third">
  <img src = "/Users/xijinping/Downloads/crawl/captcha/captcha39.png">
  <img src = "/Users/xijinping/Downloads/crawl/captcha/captcha40.png">  
  <img src = "/Users/xijinping/Downloads/crawl/captcha/captcha41.png">
</figure>
</center>

一开始的思路当然是看看有没有现成的库去处理，试用了采用OSR方法的pytesseract库，结果惨不忍睹，毕竟OSR是为了处理规则的图像中文字。那就只能采用自己标注数据+卷积神经网络训练的方法了。关于CNN，安利一篇自家学校的[CNN文章](/Users/xijinping/Downloads/gitwork/YfYan.github.io/CNN.pdf)，零基础也可以看个大概，了解大致原理有利于调参。

首先要决定的问题上是验证码是整体预测还是切割后单个预测。简单的查阅要获得足够高的准确率，整体预测需要2万左右的训练集，事实上，1000个训练集我都标了一下午，我觉得标2万个不太现实，因此采取了分割后单个字符预测的方法。切割程序为crop_captcha.py

虽然验证码的图片都是60行150列的图片，但实际的验证码大约都是60行120列，如果直接平均分割成四份非常不准确。这里采用的策略是首先将图片灰度化，只留一个颜色通道，从左到右扫描，遇到第一个黑像素点设为起点，再从右到左扫描，遇到第一个黑像素点设为终点。把起点到终点区域的图像缩放成120列，之后再平均分成四份，目测结果还行，部分切割结果如下：

![pic7](/Users/xijinping/Downloads/gitwork/YfYan.github.io/images/pic7.png)

之后的预测目标为，输入一个60 * 30 * 1的矩阵，经过神经网络和softmax层后返回一个one-hot label对应一个字符。在python中有很多机器学习的库，如果要快速实现想法的话，首推[Keras](https://keras.io/)，它把tensorflow作为backend，很多细节都隐藏了，除了第一层要指定数据维数外，其他层只需要指定每层的类型和activation函数即可，Keras会自动推测每层的维数。每次训练完成后，可以将整个网络存在本地，最后选择一个测试集效果最好的用于预测即可。本人最后采用的网络结构为：

```python
from keras.models import Sequential
from keras.layers import Conv2D,MaxPooling2D,Activation,Dense,Flatten
model=Sequential()#Sequential是Keras搭建神经网络的基本单位
model.add(Conv2D(filters=32,kernel_size=(5,5),activation='relu',input_shape=(60,30,1)))#加卷积层
model.add(MaxPooling2D(pool_size=(2,2),strides=(2,2)))#加最大值池化层
model.add(Conv2D(filters=64,kernel_size=(5,5),activation='relu'))
model.add(MaxPooling2D(pool_size=(2,2),strides=(2,2)))
model.add(Flatten())
model.add(Dense(64,activation='relu'))#加全联接层
model.add(Activation('relu'))
model.add(Dense(num_class,activation='softmax'))#softmax层异常重要
```

调参主要能调的部分有：加多少层卷积层和池化层，filters和kernal_size，filters决定了下一层神经网络接受的图层数量，kernal_size是每次卷积或池化对多少像素做处理。具体调法是玄学，个人不是这方面的专家，也只能用控制变量法多试试了，最后要用的权重及网络结构存在cnn_captcha_v4.h5中，训练所在的文件为cnn_captcha.py



最后在captcha_predict.py中封装了一个Captcha_pred类，它会加载cnn_captcha_v4.h5中的网络，然后设定验证码图片的路径，再预测一个完整的结果(它会在内部完成分割)。由于单个预测的成功率约为90%，假设独立同分布，那么四个预测的成功率约为60%，有多线程的话，只要Chrome加载出来，几乎立刻就能预测出来。多线程还有其他的坑，Keras的后端tensorflow的session和graph在多线程会出问题，查了stackoverflow后，需要在Captcha_pred初始化化有所修改:

```python
import tensorflow as tf
from keras import backend as K
class Captcha_pred(object):#其他细节都忽略了 只写了多线程的坑
  def __init__(self):
    self.session = K.get_session()
    self.graph = tf.get_default_graph()
```

另外多线程时，验证码图片和分割后图片的命名都采用了随机数，这样可以防止命名冲突。



# version 2.0

在1.2版本后