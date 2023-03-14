# 用 Python 和 Selenium 构建并发 Web Scraper

> 原文：<https://testdriven.io/blog/building-a-concurrent-web-scraper-with-python-and-selenium/>

本文着眼于如何通过`concurrent.futures`模块用多线程加速 Python web 抓取和爬行脚本。我们还将分解脚本本身，并展示如何用 pytest 测试解析功能。

完成本文后，您将能够:

1.  用 Selenium 抓取网站，用美汤解析 HTML
2.  设置 pytest 来测试抓取和解析功能
3.  与`concurrent.futures`模块同时执行网络刮刀
4.  使用 Selenium 为 ChromeDriver 配置无头模式

## 项目设置

如果你想继续的话，复制回购协议。从命令行运行以下命令:

```
`$ git clone [[email protected]](/cdn-cgi/l/email-protection):testdrivenio/concurrent-web-scraping.git
$ cd concurrent-web-scraping

$ python -m venv env
$ source env/bin/activate
(env)$ pip install -r requirements.txt` 
```

> 根据您的环境，上述命令可能会有所不同。

全局安装 [ChromeDriver](https://sites.google.com/chromium.org/driver/) 。(我们使用的是版本 [96.0.4664.45](https://chromedriver.storage.googleapis.com/index.html?path=96.0.4664.45/) )。

## 脚本概述

该脚本向 [Wikipedia:Random](https://en.wikipedia.org/wiki/Wikipedia:Random) - `https://en.wikipedia.org/wiki/Special:Random`发出 20 个请求，请求关于每篇文章的信息，使用 [Selenium](http://www.seleniumhq.org/projects/webdriver/) 自动与站点交互，使用 [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/) 解析 HTML。

*script.py* :

```
`import datetime
import sys
from time import sleep, time

from scrapers.scraper import connect_to_base, get_driver, parse_html, write_to_file

def run_process(filename, browser):
    if connect_to_base(browser):
        sleep(2)
        html = browser.page_source
        output_list = parse_html(html)
        write_to_file(output_list, filename)
    else:
        print("Error connecting to Wikipedia")

if __name__ == "__main__":

    # headless mode?
    headless = False
    if len(sys.argv) > 1:
        if sys.argv[1] == "headless":
            print("Running in headless mode")
            headless = True

    # set variables
    start_time = time()
    current_attempt = 1
    output_timestamp = datetime.datetime.now().strftime("%Y%m%d%H%M%S")
    output_filename = f"output_{output_timestamp}.csv"

    # init browser
    browser = get_driver(headless=headless)

    # scrape and crawl
    while current_attempt <= 20:
        print(f"Scraping Wikipedia #{current_attempt} time(s)...")
        run_process(output_filename, browser)
        current_attempt = current_attempt + 1

    # exit
    browser.quit()
    end_time = time()
    elapsed_time = end_time - start_time
    print(f"Elapsed run time: {elapsed_time} seconds")` 
```

让我们从主块开始。在确定 Chrome 是否应该在 headless 模式下运行并定义一些变量后，浏览器通过 *scrapers/scraper.py* 中的`get_driver()`进行初始化:

```
`if __name__ == "__main__":

    # headless mode?
    headless = False
    if len(sys.argv) > 1:
        if sys.argv[1] == "headless":
            print("Running in headless mode")
            headless = True

    # set variables
    start_time = time()
    current_attempt = 1
    output_timestamp = datetime.datetime.now().strftime("%Y%m%d%H%M%S")
    output_filename = f"output_{output_timestamp}.csv"

    ########
    # here #
    ########
    # init browser
    browser = get_driver(headless=headless)

    # scrape and crawl
    while current_attempt <= 20:
        print(f"Scraping Wikipedia #{current_attempt} time(s)...")
        run_process(output_filename, browser)
        current_attempt = current_attempt + 1

    # exit
    browser.quit()
    end_time = time()
    elapsed_time = end_time - start_time
    print(f"Elapsed run time: {elapsed_time} seconds")` 
```

然后配置一个`while`回路来控制整个刮刀的流量。

```
`if __name__ == "__main__":

    # headless mode?
    headless = False
    if len(sys.argv) > 1:
        if sys.argv[1] == "headless":
            print("Running in headless mode")
            headless = True

    # set variables
    start_time = time()
    current_attempt = 1
    output_timestamp = datetime.datetime.now().strftime("%Y%m%d%H%M%S")
    output_filename = f"output_{output_timestamp}.csv"

    # init browser
    browser = get_driver(headless=headless)

    ########
    # here #
    ########
    # scrape and crawl
    while current_attempt <= 20:
        print(f"Scraping Wikipedia #{current_attempt} time(s)...")
        run_process(output_filename, browser)
        current_attempt = current_attempt + 1

    # exit
    browser.quit()
    end_time = time()
    elapsed_time = end_time - start_time
    print(f"Elapsed run time: {elapsed_time} seconds")` 
```

在这个循环中，`run_process()`被调用，它管理 [WebDriver](https://developer.mozilla.org/en-US/docs/Web/WebDriver) 的连接和抓取功能。

```
`def run_process(filename, browser):
    if connect_to_base(browser):
        sleep(2)
        html = browser.page_source
        output_list = parse_html(html)
        write_to_file(output_list, filename)
    else:
        print("Error connecting to Wikipedia")` 
```

在`run_process()`中，浏览器实例传递给了`connect_to_base()`。

```
`def run_process(filename, browser):

    ########
    # here #
    ########
    if connect_to_base(browser):
        sleep(2)
        html = browser.page_source
        output_list = parse_html(html)
        write_to_file(output_list, filename)
    else:
        print("Error connecting to wikipedia")` 
```

这个函数试图连接到 wikipedia，然后使用 Selenium 的显式等待功能来确保带有`id='content'`的元素在继续之前已经加载。

```
`def connect_to_base(browser):
    base_url = "https://en.wikipedia.org/wiki/Special:Random"
    connection_attempts = 0
    while connection_attempts < 3:
        try:
            browser.get(base_url)
            # wait for table element with id = 'content' to load
            # before returning True
            WebDriverWait(browser, 5).until(
                EC.presence_of_element_located((By.ID, "content"))
            )
            return True
        except Exception as e:
            print(e)
            connection_attempts += 1
            print(f"Error connecting to {base_url}.")
            print(f"Attempt #{connection_attempts}.")
    return False` 
```

> 查看 Selenium [文档](http://selenium-python.readthedocs.io/waits.html#explicit-waits)了解更多关于显式等待的信息。

为了模拟人类用户，在浏览器连接到维基百科后会调用`sleep(2)`。

```
`def run_process(filename, browser):
    if connect_to_base(browser):

        ########
        # here #
        ########
        sleep(2)
        html = browser.page_source
        output_list = parse_html(html)
        write_to_file(output_list, filename)
    else:
        print("Error connecting to Wikipedia")` 
```

一旦页面被加载并且`sleep(2)`被执行，浏览器获取 HTML 源，然后传递给`parse_html()`。

```
`def run_process(filename, browser):
    if connect_to_base(browser):
        sleep(2)

        ########
        # here #
        ########
        html = browser.page_source

        ########
        # here #
        ########
        output_list = parse_html(html)
        write_to_file(output_list, filename)
    else:
        print("Error connecting to Wikipedia")` 
```

使用 Beautiful Soup 解析 HTML，生成包含适当数据的字典列表。

```
`def parse_html(html):
    # create soup object
    soup = BeautifulSoup(html, "html.parser")
    output_list = []
    # parse soup object to get wikipedia article url, title, and last modified date
    article_url = soup.find("link", {"rel": "canonical"})["href"]
    article_title = soup.find("h1", {"id": "firstHeading"}).text
    article_last_modified = soup.find("li", {"id": "footer-info-lastmod"}).text
    article_info = {
        "url": article_url,
        "title": article_title,
        "last_modified": article_last_modified,
    }
    output_list.append(article_info)
    return output_list` 
```

该函数还将文章 URL 传递给`get_load_time()`，由它加载 URL 并记录随后的加载时间。

```
`def get_load_time(article_url):
    try:
        # set headers
        headers = {
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.89 Safari/537.36"
        }
        # make get request to article_url
        response = requests.get(
            article_url, headers=headers, stream=True, timeout=3.000
        )
        # get page load time
        load_time = response.elapsed.total_seconds()
    except Exception as e:
        print(e)
        load_time = "Loading Error"
    return load_time` 
```

输出被添加到 CSV 文件中。

```
`def run_process(filename, browser):
    if connect_to_base(browser):
        sleep(2)
        html = browser.page_source
        output_list = parse_html(html)

        ########
        # here #
        ########
        write_to_file(output_list, filename)
    else:
        print("Error connecting to Wikipedia")` 
```

`write_to_file()`:

```
`def write_to_file(output_list, filename):
    for row in output_list:
        with open(Path(BASE_DIR).joinpath(filename), "a") as csvfile:
            fieldnames = ["url", "title", "last_modified"]
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
            writer.writerow(row)` 
```

最后，回到`while`循环中，`current_attempt`递增，过程再次开始。

```
`if __name__ == "__main__":

    # headless mode?
    headless = False
    if len(sys.argv) > 1:
        if sys.argv[1] == "headless":
            print("Running in headless mode")
            headless = True

    # set variables
    start_time = time()
    current_attempt = 1
    output_timestamp = datetime.datetime.now().strftime("%Y%m%d%H%M%S")
    output_filename = f"output_{output_timestamp}.csv"

    # init browser
    browser = get_driver(headless=headless)

    # scrape and crawl
    while current_attempt <= 20:
        print(f"Scraping Wikipedia #{current_attempt} time(s)...")
        run_process(output_filename, browser)

        ########
        # here #
        ########
        current_attempt = current_attempt + 1

    # exit
    browser.quit()
    end_time = time()
    elapsed_time = end_time - start_time
    print(f"Elapsed run time: {elapsed_time} seconds")` 
```

> 想测试一下吗？点击获取完整脚本[。](https://github.com/testdrivenio/concurrent-web-scraping/blob/master/script.py)

运行大约需要 57 秒:

```
`(env)$ python script.py

Scraping Wikipedia #1 time(s)...
Scraping Wikipedia #2 time(s)...
Scraping Wikipedia #3 time(s)...
Scraping Wikipedia #4 time(s)...
Scraping Wikipedia #5 time(s)...
Scraping Wikipedia #6 time(s)...
Scraping Wikipedia #7 time(s)...
Scraping Wikipedia #8 time(s)...
Scraping Wikipedia #9 time(s)...
Scraping Wikipedia #10 time(s)...
Scraping Wikipedia #11 time(s)...
Scraping Wikipedia #12 time(s)...
Scraping Wikipedia #13 time(s)...
Scraping Wikipedia #14 time(s)...
Scraping Wikipedia #15 time(s)...
Scraping Wikipedia #16 time(s)...
Scraping Wikipedia #17 time(s)...
Scraping Wikipedia #18 time(s)...
Scraping Wikipedia #19 time(s)...
Scraping Wikipedia #20 time(s)...
Elapsed run time: 57.36561393737793 seconds` 
```

明白了吗？太好了！让我们添加一些基本的测试。

## 测试

要在不启动浏览器的情况下测试解析功能，从而向 Wikipedia 发出重复的 GET 请求，您可以下载页面的 HTML ( *test/test.html* )并在本地解析它。这有助于避免在编写和测试解析函数时，由于太快发出太多请求而导致 IP 被阻塞的情况，同时也节省了时间，因为不必在每次运行脚本时都打开浏览器。

*test/test_scraper.py* :

```
`from pathlib import Path

import pytest

from scrapers import scraper

BASE_DIR = Path(__file__).resolve(strict=True).parent

@pytest.fixture(scope="module")
def html_output():
    with open(Path(BASE_DIR).joinpath("test.html"), encoding="utf-8") as f:
        html = f.read()
        yield scraper.parse_html(html)

def test_output_is_not_none(html_output):
    assert html_output

def test_output_is_a_list(html_output):
    assert isinstance(html_output, list)

def test_output_is_a_list_of_dicts(html_output):
    assert all(isinstance(elem, dict) for elem in html_output)` 
```

确保一切正常:

```
`(env)$ python -m pytest test/test_scraper.py

================================ test session starts =================================
platform darwin -- Python 3.10.0, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
rootdir: /Users/michael/repos/testdriven/async-web-scraping
collected 3 items

test/test_scraper.py ...                                                       [100%]

================================= 3 passed in 0.19 ==================================` 
```

想模仿`get_load_time()`绕过 GET 请求？

*test/test _ scraper _ mock . py*:

```
`from pathlib import Path

import pytest

from scrapers import scraper

BASE_DIR = Path(__file__).resolve(strict=True).parent

@pytest.fixture(scope="function")
def html_output(monkeypatch):
    def mock_get_load_time(url):
        return "mocked!"

    monkeypatch.setattr(scraper, "get_load_time", mock_get_load_time)
    with open(Path(BASE_DIR).joinpath("test.html"), encoding="utf-8") as f:
        html = f.read()
        yield scraper.parse_html(html)

def test_output_is_not_none(html_output):
    assert html_output

def test_output_is_a_list(html_output):
    assert isinstance(html_output, list)

def test_output_is_a_list_of_dicts(html_output):
    assert all(isinstance(elem, dict) for elem in html_output)` 
```

测试:

```
`(env)$ python -m pytest test/test_scraper_mock.py

================================ test session starts =================================
platform darwin -- Python 3.10.0, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
rootdir: /Users/michael/repos/testdriven/async-web-scraping
collected 3 items

test/test_scraper.py ...                                                       [100%]

================================= 3 passed in 0.27s =================================` 
```

## 配置多线程

现在有趣的部分来了！通过对脚本进行一些修改，我们可以加快速度:

```
`import datetime
import sys
from concurrent.futures import ThreadPoolExecutor, wait
from time import sleep, time

from scrapers.scraper import connect_to_base, get_driver, parse_html, write_to_file

def run_process(filename, headless):

    # init browser
    browser = get_driver(headless)

    if connect_to_base(browser):
        sleep(2)
        html = browser.page_source
        output_list = parse_html(html)
        write_to_file(output_list, filename)

        # exit
        browser.quit()
    else:
        print("Error connecting to Wikipedia")
        browser.quit()

if __name__ == "__main__":

    # headless mode?
    headless = False
    if len(sys.argv) > 1:
        if sys.argv[1] == "headless":
            print("Running in headless mode")
            headless = True

    # set variables
    start_time = time()
    output_timestamp = datetime.datetime.now().strftime("%Y%m%d%H%M%S")
    output_filename = f"output_{output_timestamp}.csv"
    futures = []

    # scrape and crawl
    with ThreadPoolExecutor() as executor:
        for number in range(1, 21):
            futures.append(
                executor.submit(run_process, output_filename, headless)
            )

    wait(futures)
    end_time = time()
    elapsed_time = end_time - start_time
    print(f"Elapsed run time: {elapsed_time} seconds")` 
```

有了`concurrent.futures`库，`ThreadPoolExecutor`被用来产生一个线程池来异步执行`run_process`函数。[提交](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Executor.submit)方法接受该函数及其参数，并返回一个[未来](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Future)对象。 [wait](https://docs.python.org/3.4/library/concurrent.futures.html#module-functions) 用于阻塞执行，直到所有任务完成。

值得注意的是，您可以通过`ProcessPoolExecutor`轻松切换到多处理，因为`ProcessPoolExecutor`和`ThreadPoolExecutor`实现了相同的接口:

```
`# scrape and crawl
with ProcessPoolExecutor() as executor:
    for number in range(1, 21):
        futures.append(
            executor.submit(run_process, output_filename, headless)
        )` 
```

> 为什么是多线程而不是多重处理？
> 
> Web 抓取是 I/O 绑定的，因为获取 HTML (I/O)比解析它(CPU)慢。关于这一点以及并行性(多处理)和并发性(多线程)之间的区别的更多信息，请回顾文章[用并发性、并行性和异步性加速 Python。](/blog/concurrency-parallelism-asyncio/)

运行:

```
`(env)$ python script_concurrent.py

Elapsed run time: 11.831077098846436 seconds` 
```

> 点击查看完整的脚本[。](https://github.com/testdrivenio/concurrent-web-scraping/blob/master/script_concurrent.py)

为了进一步加快速度，我们可以通过传入`headless`命令行参数在无头模式下运行 Chrome:

```
`(env)$ python script_concurrent.py headless

Running in headless mode
Elapsed run time: 6.222846269607544 seconds` 
```

## 结论

通过对原始代码进行少量修改，我们能够并发执行 web scraper，将脚本的运行时间从大约 57 秒缩短到 6 秒多一点。在这个特定场景中，速度提高了大约 90%，这是一个巨大的进步。

我希望这对你的剧本有所帮助。你可以在[回购](https://github.com/testdrivenio/concurrent-web-scraping/)中找到代码。干杯！