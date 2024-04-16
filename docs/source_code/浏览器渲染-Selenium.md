# 浏览器渲染-Selenium

采集动态页面时（Ajax渲染的页面），常用的有两种方案。一种是找接口拼参数，这种方式比较复杂但效率高，需要一定的爬虫功底；另外一种是采用浏览器渲染的方式，直接获取源码，简单方便

框架内置一个浏览器渲染池，默认的池子大小为1，请求时重复利用浏览器实例，只有当代理失效请求异常时，才会销毁、创建一个新的浏览器实例

内置浏览器渲染支持 **CHROME**、**EDGE**、**PHANTOMJS**、**FIREFOX**

## 使用方式：

```python
def start_requests(self):
    yield feapder.Request("https://news.qq.com/", render=True)
```
在返回的Request中传递`render=True`即可

框架支持`CHROME`、`EDGE`、`PHANTOMJS`、`FIREFOX` 三种浏览器渲染，可通过[配置文件](source_code/配置文件)进行配置。相关配置如下：

```python
# 浏览器渲染
WEBDRIVER = dict(
    pool_size=1,  # 浏览器的数量
    load_images=True,  # 是否加载图片
    user_agent=None,  # 字符串 或 无参函数，返回值为user_agent
    proxy=None,  # xxx.xxx.xxx.xxx:xxxx 或 无参函数，返回值为代理地址
    headless=False,  # 是否为无头浏览器
    driver_type="CHROME",  # CHROME、EDGE、PHANTOMJS、FIREFOX
    timeout=30,  # 请求超时时间
    window_size=(1024, 800),  # 窗口大小
    executable_path=None,  # 浏览器路径，默认为默认路径
    render_time=0, # 渲染时长，即打开网页等待指定时间后再获取源码
    custom_argument=["--ignore-certificate-errors"],  # 自定义浏览器渲染参数
    xhr_url_regexes=None,  # 拦截xhr接口，支持正则，数组类型
    auto_install_driver=False,  # 自动下载浏览器驱动 支持chrome 和 firefox
)
```

 - `feapder.Request` 也支持`render_time`参数， 优先级大于配置文件中的`render_time`

 - 代理使用优先级：`feapder.Request`指定的代理 > 配置文件中的`PROXY_EXTRACT_API` > webdriver配置文件中的`proxy`

 - user_agent使用优先级：`feapder.Request`指定的header里的`User-Agent` > 框架随机的`User-Agent` > webdriver配置文件中的`user_agent`

## 设置User-Agent

> 每次生成一个新的浏览器实例时生效

### 方式1：

通过配置文件的 `user_agent` 参数设置

### 方式2：

通过 `feapder.Request`携带，优先级大于配置文件, 如：

```python
def download_midware(self, request):
    request.headers = {
        "User-Agent": "xxxxxxxx"
    }
    return request
```

## 设置代理

> 每次生成一个新的浏览器实例时生效

### 方式1：

通过配置文件的 `proxy` 参数设置

### 方式2：

通过 `feapder.Request`携带，优先级大于配置文件, 如：

```python
def download_midware(self, request):
    request.proxies = {
        "https": "https://xxx.xxx.xxx.xxx:xxxx"
    }
    return request
```

## 设置Cookie

通过 `feapder.Request`携带，如：

```python
def download_midware(self, request):
    request.headers = {
        "Cookie": "key=value; key2=value2"
    }
    return request
```

或者

```python
def download_midware(self, request):
    request.cookies = {
        "key": "value",
        "key2": "value2",
    }
    return request
```

或者

```python
def download_midware(self, request):
    request.cookies = [
        {
            "domain": "xxx",
            "name": "xxx",
            "value": "xxx",
            "expirationDate": "xxx"
        },
    ]
    return request
```

## 操作浏览器对象

通过 `response.browser` 获取浏览器对象

代码示例：请求百度，搜索feapder

```python
import time

import feapder
from feapder.utils.webdriver import WebDriver


class TestRender(feapder.AirSpider):
    def start_requests(self):
        yield feapder.Request("http://www.baidu.com", render=True)

    def parse(self, request, response):
        browser: WebDriver = response.browser
        browser.find_element_by_id("kw").send_keys("feapder")
        browser.find_element_by_id("su").click()
        time.sleep(5)
        print(browser.page_source)

        # response也是可以正常使用的
        # response.xpath("//title")

        # 若有滚动，可通过如下方式更新response，使其加载滚动后的内容
        # response.text = browser.page_source


if __name__ == "__main__":
    TestRender().start()

```

## 拦截xhr数据

### 设置拦截规则

```python
WEBDRIVER = dict(
    ...
    xhr_url_regexes=[
        "接口1正则",
        "接口2正则",
    ]
)
```

### 获取数据

```python
browser: WebDriver = response.browser
text = browser.xhr_text("接口1正则")
```

### 获取json格式数据

```python
browser: WebDriver = response.browser
data = browser.xhr_json("接口1正则")
```

### 获取response

```python
browser: WebDriver = response.browser
xhr_response = browser.xhr_response("接口1正则")
print("请求接口", xhr_response.request.url)
print("请求头", xhr_response.request.headers)
print("请求体", xhr_response.request.data)
print("返回头", xhr_response.headers)
print("返回地址", xhr_response.url)
print("返回内容", xhr_response.content)
```


### 示例：

需求：
![](http://markdown-media.oss-cn-beijing.aliyuncs.com/2021/12/30/16408610725756.jpg)

代码：

```python
import time

import feapder
from feapder.utils.webdriver import WebDriver


class TestRender(feapder.AirSpider):
    __custom_setting__ = dict(
        WEBDRIVER=dict(
            pool_size=1,  # 浏览器的数量
            load_images=True,  # 是否加载图片
            user_agent=None,  # 字符串 或 无参函数，返回值为user_agent
            proxy=None,  # xxx.xxx.xxx.xxx:xxxx 或 无参函数，返回值为代理地址
            headless=False,  # 是否为无头浏览器
            driver_type="CHROME",  # CHROME、EDGE、PHANTOMJS、FIREFOX
            timeout=30,  # 请求超时时间
            window_size=(1024, 800),  # 窗口大小
            executable_path=None,  # 浏览器路径，默认为默认路径
            render_time=0,  # 渲染时长，即打开网页等待指定时间后再获取源码
            custom_argument=["--ignore-certificate-errors"],  # 自定义浏览器渲染参数
            xhr_url_regexes=[
                "/ad",
            ],  # 拦截 http://www.spidertools.cn/spidertools/ad 接口
        )
    )

    def start_requests(self):
        yield feapder.Request("http://www.spidertools.cn", render=True)

    def parse(self, request, response):
        browser: WebDriver = response.browser
        time.sleep(3)

        # 获取接口数据 文本类型
        ad = browser.xhr_text("/ad")
        print(ad)

        # 获取接口数据 转成json，本例因为返回的接口是文本，所以不转了
        # browser.xhr_json("/ad")

        xhr_response = browser.xhr_response("/ad")
        print("请求接口", xhr_response.request.url)
        # 请求头目前获取的不完整
        print("请求头", xhr_response.request.headers)
        print("请求体", xhr_response.request.data)
        print("返回头", xhr_response.headers)
        print("返回地址", xhr_response.url)
        print("返回内容", xhr_response.content)


if __name__ == "__main__":
    TestRender().start()

```

## 驱动版本自动适配

```python
WEBDRIVER = dict(
    ...
    auto_install_driver=True
)
```

即浏览器渲染相关配置 `auto_install_driver` 设置True，让其自动对比驱动版本，版本不符或驱动不存在时自动下载

## 关闭当前浏览器

```python
def parse(self, request, response):
    response.close_browser(request)
```

关闭会自动重开一个新的浏览器实例
