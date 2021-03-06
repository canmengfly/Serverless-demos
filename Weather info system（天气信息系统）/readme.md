## 云函数 + API，你也可以做个天气信息系统

通常，我们用云函数 SCF 写一个函数应用，这个应用可能多种多样。例如之前介绍过的 [OJ 系统判题功能](https://zhuanlan.zhihu.com/p/82651235)，[通过 NLP 实现文本摘要功能](https://zhuanlan.zhihu.com/p/78336933)......，那么，怎么把这些功能简单快速地结合到我们的项目中，尤其是 Web 项目中呢？

本文通过一个简单小例子实现云函数 SCF 与 API 网关的结合，希望能给到大家一个参考。

## **▎任务说明**

通过云函数 SCF 编写两个爬虫程序，分别是通过 IP 地址获得 IP 归属地信息、通过地址获得天气预报信息等。通过 API 网关作为触发器，实现一个简单的对外接口。

该功能主要作用是作为网站的一个接口，保证用户访问网站时，可以在适当的位置看到今天本地区的天气信息。

## **▎任务流程**

![img](https://pic2.zhimg.com/80/v2-5ace189cfe0f0424f2d161ea67cfa1c9_hd.jpg)

## **▎爬虫实现**

### **爬虫 1 实现：获得 IP 地址**

搜索 IP 地址，可以看到这样一个小工具：

![img](https://pic3.zhimg.com/80/v2-aaca0ab3517ea1167312260f3d7dcaea_hd.jpg)

输入 IP 地址，点击查询可以获得到地址信息。通过抓包可以获得 API：

```text
https://sp0.baidu.com/8aQDcjqpAAV3otqbppnN2DJv/api.php?query=113.57.215.184&co=&resource_id=6006&t=1559922221313&ie=utf8&oe=gbk&cb=op_aladdin_callback&format=json&tn=baidu&cb=jQuery110205516131051897397_1559921486295&_=1559921486372
```

结果如下：

![img](https://pic1.zhimg.com/80/v2-16776b5c6fc7faa11ce1acde60775510_hd.png)

对地址进行简化：

```text
https://sp0.baidu.com/8aQDcjqpAAV3otqbppnN2DJv/api.php?query=113.57.215.184&resource_id=6006&format=json
```

简化后结果成为 Json 形式：

![img](https://pic1.zhimg.com/80/v2-dec0249401d66babb174ab9446cc7ed4_hd.png)

编写 Python 代码实现：

```text
import urllib.request
import ssl
import json
ssl._create_default_https_context = ssl._create_unverified_context
location_temp = json.loads(urllib.request.urlopen(
    "https://sp0.baidu.com/8aQDcjqpAAV3otqbppnN2DJv/api.php?query=113.57.215.184&resource_id=6006&format=json").read().decode(
    "gbk"))["data"][0]["location"]
location = location_temp.split(" ")[0] if " " in location_temp else location_temp
print(location)
```

运行结果：

![img](https://pic3.zhimg.com/80/v2-c89185f10ce87dc96f5cbdaae856d99e_hd.jpg)

### **爬虫 2 实现：获取天气**

搜索天气，可以得到：

![img](https://pic2.zhimg.com/80/v2-07e928d7c75f14abbe28c5386006d3fd_hd.jpg)

对页面分析，我们可以看到天气信息在网页源码中可以提现：

![img](https://pic4.zhimg.com/80/v2-a65e612fedf2d5517d924697afeb96a7_hd.jpg)

也就是说，我们可以通过简单的页面分析，就能获得到天气数据：

```text
import urllib.request
import urllib.parse
url = "http://www.baidu.com/s?wd=" + urllib.parse.quote("湖北省武汉市天气")
page_source = urllib.request.urlopen(url).read().decode("utf-8").replace("\n", "").replace("\r", "")
weather = page_source.split('<p class="op_weather4_twoicon_weath"')[1].split('title="">')[1].split('</p>')[0].strip()
temp = page_source.split('<p class="op_weather4_twoicon_temp">')[1].split('</p>')[0].strip()
print(weather,temp)
```

运行结果：

![img](https://pic3.zhimg.com/80/v2-57b4ece52f2b908a785c058967f340ee_hd.png)

## **▎云函数 API 网关触发器**

新建云函数：

![img](https://pic3.zhimg.com/80/v2-2e672aabf9edce4c4ccc230f49c75ac6_hd.jpg)

保存之后，在测试的时候，选择 API 网关作为触发器，进行测试：

![img](https://pic2.zhimg.com/80/v2-5f830b408e08321d1046f94c306f26d1_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-8fbb7f85af02e371bb8cd528245b1c9e_hd.jpg)

测试之后，可以看到结果，便于我们进行基本分析：

![img](https://pic3.zhimg.com/80/v2-f4a138192b67cd36c52c5f05589d54b6_hd.jpg)

经过分析可以看到 Event 中有：

![img](https://pic1.zhimg.com/80/v2-52661d1bb4e892b8623a1e171db9edbc_hd.jpg)

所以，我们可以获得这个 IP 地址：

```text
# -*- coding: utf8 -*-
import json
def main_handler(event, context):
    print(event["requestContext"]["sourceIp"])
```

运行结果：

![img](https://pic2.zhimg.com/80/v2-5f1b1f35be21c0237bdc5bff387467f9_hd.jpg)

## **▎代码整合**

```text
# -*- coding: utf8 -*-
import json, ssl
import urllib.request
import urllib.parse

ssl._create_default_https_context = ssl._create_unverified_context

def get_loaction(ip):
    location_temp = json.loads(urllib.request.urlopen("https://sp0.baidu.com/8aQDcjqpAAV3otqbppnN2DJv/api.php?query=" + ip + "&resource_id=6006&format=json").read().decode("gbk"))["data"][0]["location"]
    return location_temp.split(" ")[0] if " " in location_temp else location_temp

def get_weather(address):
    url = "http://www.baidu.com/s?wd=" + urllib.parse.quote(address + "天气")
    page_source = urllib.request.urlopen(url).read().decode("utf-8").replace("\n", "").replace("\r", "")
    weather = page_source.split('<p class="op_weather4_twoicon_weath"')[1].split('title="">')[1].split('</p>')[0].strip()
    temp = page_source.split('<p class="op_weather4_twoicon_temp">')[1].split('</p>')[0].strip()
    return {"weather": weather, "temp": temp}

def main_handler(event, context):
    return get_weather(get_loaction(event["requestContext"]["sourceIp"]))
```

测试结果：

![img](https://pic1.zhimg.com/80/v2-4edabc548cfc7abec056da55ab393abc_hd.jpg)

## **▎结合 API 网关**

选择 API 网关：

![img](https://pic1.zhimg.com/80/v2-a2d226f8dfbfdc197316bd38e7d68884_hd.jpg)

在与云函数相同区域，建立：

![img](https://pic3.zhimg.com/80/v2-14801199c98f00ca4abb82a955f377a6_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-c8a53f84d86631504b2311f040f89465_hd.jpg)

保存之后会提示我们进行 API 配置：

![img](https://pic1.zhimg.com/80/v2-9f71e91ea3ab36da2dcaf48383a1ff88_hd.jpg)

点击新建：

![img](https://pic4.zhimg.com/80/v2-2e8a6c0fa82ee4e3fb0f0e2c8c778303_hd.jpg)

因为本文仅是做一个简单的 Demo。所以此处我们只进行简单配置，例如鉴权等都选择了免鉴权，但是在实际中还是推荐大家，进行鉴权，这样更安全，也避免资源被盗用等，除此之外，其他各个参数都需要根据自己需求而定：

![img](https://pic2.zhimg.com/80/v2-7c55bafaaee71ad4f56f098374bcfa39_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-d35fd57341c57e30842def7fe6adfd27_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-425b994575732573aeb4c1cabb896a23_hd.jpg)

配置完成之后，发布测试环境进行测试：

![img](https://pic1.zhimg.com/80/v2-813c8273843ea68ca1b4f0aeb89bd488_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-7d259dd7e63de4d157624a8684e34ad3_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-96efeda6ebefa60e08abfd8a4947df6c_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-86d3ea5d8688092e7785ec151fd6535d_hd.jpg)

测试发布完成之后，我们通过浏览器进行一下简单测试：

复制地址，并添加我们之前的路径：

![img](https://pic2.zhimg.com/80/v2-7b9138e69b6700516f5b45a5990cac9d_hd.png)

至此，我们就完成了一个 API 网关与 SCF 结合的小例子。

## **▎额外想说**

云函数是一个函数级别的应用，我们可以将它应用在很多领域，例如 Web 开发、Iot 等。但是云函数本身自己很难完成一个功能，它需要和周边的产品配合，例如和 [COS](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/product/cos%3Ffrom%3D9253) 配合实现图像压缩/加水印，本文则是说与 API 网关结合做一个获取天气的 HTTP 接口。其实大家还可以想一下，我们是不是可以通过 SCF 与 API 网关结合，实现一个 Web 后端呢？

**以一个博客系统为例：**

前端使用 Vue.js 等框架进行开发，所有的后端逻辑，包括数据库的增删改查、某些小功能点的实现，全部用云函数来实现？这样，我们只需要找一个虚拟空间或者腾讯云的 COS，就可以完成前端的部署，而后端的服务器配置、面对用户激增的服务器运维等，都交给云函数+相关产品来实现，是不是大大节约资源，降低成本呢？

总的来说，合理利用云函数，能够节约资源、降低成本、提高效率。

> 本文亦发布于[知乎](https://zhuanlan.zhihu.com/p/83753850)