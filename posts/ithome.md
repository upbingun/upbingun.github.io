---
title: IT之家新闻API以及实战
tags: [API]
categories: [API]
date: 2025-10-25
description: 使用IT之家的API并利用Python进行新闻打印。
articleGPT: 这是一篇有关IT之家新闻获取文章，旨在于介绍IT之家的API
references:
  - title: vitepress-theme-curve
    url: https://github.com/imsyy/vitepress-theme-curve
---

# IT之家新闻API以及实战
欢迎来到upbingun的博客。<br>
我们将会在此介绍IT之家的一些API，以及如何用它制作一个应用程序。<br>
:::info
考虑到种种因素，本次示例使用Python。因为我并不专业Python所以这些代码是由AI生成的。
:::

## 起因
我之前搞了个NOKIA 8110用来打电话（HMD时代的手机，使用KaiOS，后来在[诺亚方舟号](https://www.dospy.wang/)看到有一些搞机文章装应用进去，于是看到有个IT之家的KaiOS版（文章：[原文链接](https://www.dospy.wang/thread-13183-1-1.html)）。除了按键粘连回导致有时候不小心退出其他都还是蛮不错的。正好以此机会认识到了IT之家这个新闻网站，个人认为新闻还是蛮不错的。<br>
既然都能在KaiOS上适配，那肯定有比较简易的API用于获取新闻的。

## API的收集
### 在网上的搜寻
在Bing/Google上搜索`IT之家API`可以找到一个较为有用的GitHub项目，为[F-loat的ithome-lite](https://github.com/F-loat/ithome-lite)，作者在`readme.md`中写道：

>* 新闻列表 https://api.ithome.com/json/newslist/news?r=0
>* 文章详情 https://api.ithome.com/xml/newscontent/350/412.xml
>* 相关文章 https://api.ithome.com/json/tags/0350/350362.json
>* 最热评论 https://dyn.ithome.com/json/hotcommentlist/350/87a8e5b144d81938.json
>* 评论列表 https://dyn.ithome.com/json/commentlist/350/87a8e5b144d81938.json
>* 评论详情 https://dyn.ithome.com/json/commentcontent/d739ee8f2ceb0a27.json
>* 轮播新闻 https://api.ithome.com/xml/slide/slide.xml
>* 圈子列表 https://apiquan.ithome.com/api/post?categoryid=0&type=0&orderTime=&visistCount&pageLength
>* 圈子详情 https://apiquan.ithome.com/api/post/236076
>* 圈子评论 https://apiquan.ithome.com/api/reply?postid=236076&replyidlessthan=3241294

经过了我的整理，大概就是类似于`350`这种是前3位的`newsid`，`412`就是后三位的。但是评论中的`87a8e5b144d81938`这种经过我的查找发现是嵌在HTML内的（类似于`https://www.ithome.com/0/892/239.htm`这种的。在我所观察的网页源代码中（可能会过一阵被IT之家删掉）有
```html
<div id="post_comm" data-id="7afc9486b51b2e5a" data-nid="892239"></div>
```
这说明按照我的思路必须要获取这个网页本身才能得到这个`data-id`。这未免成本有点高。我在这里不做演示了，我认为总体思路可以获取源代码并提取这一字段然后扒出来。我只在这里演示获取文章内容了。<br>
但是我直接请求新闻列表的时候，发现只有第一页，没有余下的那几页，`r`的值貌似不影响页数。
### 新闻列表API的获取
我去网页上监测网络活动看看网页是怎么获取新闻列表的。在[m.ithome.com](https://m.ithome.com/)上抓包，抓到`
https://m.ithome.com/api/news/newslistpageget?Tag=&ot=1761353613000&page=0&hitCountAuthority=false`这个列表，内容为：
```json
{Success: 1,…}
Result: [{newsid: 892244, title: "HMD 首款模块化手机继任者 Fusion 2 曝光：升级骁龙 6s Gen 4 处理器、换用 FHD+ 120Hz 面板", v: null,…},…]
0: {newsid: 892244, title: "HMD 首款模块化手机继任者 Fusion 2 曝光：升级骁龙 6s Gen 4 处理器、换用 FHD+ 120Hz 面板", v: null,…}
1: {newsid: 892243, title: "里程碑：首个 SDK 发布，开发者能用苹果 Swift 语言开发安卓 App", v: null,…}
2:
More...
24: {newsid: 892223, title: "买家网购苹果手机仅退款不退货遭商家维权，法官调解后支付货款", v: "100",…}
Success: 1
```
其中`page`在网页端接下来的获取链接是没有变动的，只有`ot`变动。其中`ot`是时间戳。`ot`参数实际上是上一页最后一条新闻的 `orderdate` 时间戳。（_DeepSeek_ 指出。）所以接下来都好办了。<br>
因此构造链接大致为：
```url
https://m.ithome.com/api/news/newslistpageget?Tag=&ot={时间戳}&page=0&hitCountAuthority=false
```
### 总结
因此我们整理出来的可用API有
> * 获取新闻内容 https://api.ithome.com/xml/newscontent/前三位/后三位.xml
> * 获取文章列表 https://m.ithome.com/api/news/newslistpageget?Tag=&ot={时间戳}&page=0&hitCountAuthority=false

### API请求返回内容：
以下为我在2025-10-25 11:34请求`https://m.ithome.com/api/news/newslistpageget?Tag=&ot=1761354775000&page=0&hitCountAuthority=false`返回的内容
```json
{"Success":1,"Result":[
    {"newsid":892248,"title":"RAID 1 阵列 Win10 设备升级 Win11“变砖”，初步锁定英特尔 RST 驱动惹祸","v":null,"orderdate":"2025-10-25T09:12:52.55","postdate":"2025-10-25T09:12:52.55","description":"科技媒体 borncity 昨日（10 月 24 日）发布博文，报道称有用户反馈，在配置了 RAID 1 阵列的惠普（HP）电脑上，如果从 Windows 10 升级至 Windows 11，系统会遭遇致命的启动失败问题。","image":"https://img.ithome.com/newsuploadfiles/thumbnail/2025/10/892248_240.jpg?r=1761354772550","slink":null,"hitcount":0,"commentcount":28,"hidecount":false,"subtitle":null,"cid":183,"url":"/0/892/248.htm","live":0,"lapinid":null,"forbidcomment":false,"imagelist":null,"c":null,"client":null,"isad":false,"sid":0,"PostDateStr":"09:12","HitCountStr":null,"WapNewsUrl":"https://m.ithome.com/html/892248.htm","NewsTips":[],"IsChineseMainland":true},
    {"newsid":892247,"title":"保时捷今年前三季度营业利润 4000 万欧元暴跌 99%，三款燃油车型即将停产","v":null,"orderdate":"2025-10-25T09:10:37.187","postdate":"2025-10-25T09:10:37.187","description":"公司解释称，利润大幅下降的原因包括产品战略调整、中国市场的不利形势，以及电池相关的一次性费用。此外，关税也是一大因素，预计今年关税支出将达到 7 亿欧元。","image":"https://img.ithome.com/newsuploadfiles/thumbnail/2025/10/892247_240.jpg?r=1761354637187","slink":null,"hitcount":0,"commentcount":70,"hidecount":false,"subtitle":null,"cid":192,"url":"/0/892/247.htm","live":0,"lapinid":null,"forbidcomment":false,"imagelist":null,"c":null,"client":null,"isad":false,"sid":0,"PostDateStr":"09:10","HitCountStr":null,"WapNewsUrl":"https://m.ithome.com/html/892247.htm","NewsTips":[],"IsChineseMainland":true}...
]}
```
| 字段名                                              | 含义                                                                                          |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `newsid`                                         | 文章 ID（如 `892248`）                                                                           |
| `title`                                          | 新闻标题                                                                                        |
| `description`                                    | 摘要或导语                                                                                       |
| `image`                                          | 缩略图地址                                                                                       |
| `url`                                            | PC 网页相对路径（如 `/0/892/248.htm`）                                                               |
| `WapNewsUrl`                                     | 手机版新闻完整链接（例如：[https://m.ithome.com/html/892248.htm）](https://m.ithome.com/html/892248.htm）) |
| `commentcount`                                   | 评论数                                                                                         |
| `orderdate` / `postdate`                         | 发布时间（ISO 时间格式）                                                                              |
| `cid`                                            | 分类 ID（例如 183 可能是 Windows / 系统类新闻）                                                             |
| `NewsTips`                                       | 标签信息，如「广告」「视频」等                                                                             |
| `IsChineseMainland`                              | 是否中国大陆新闻                                                                                    |
| `title` / `description` / `image` 组合可直接用于展示新闻卡片。 |                                                                                             |

以下为我在2025-10-25 11:43请求`https://api.ithome.com/xml/newscontent/892/248.xml`得到的：
```xml
<rss version="2.0">
<channel><item><newsid>892248</newsid>
<title>
<![CDATA[ RAID 1 阵列 Win10 设备升级 Win11“变砖”，初步锁定英特尔 RST 驱动惹祸 ]]>
</title>
<c/>
<v>000</v>
<url>/0/892/248.htm</url>
<image>https://img.ithome.com/newsuploadfiles/thumbnail/2025/10/892248_240.jpg</image>
<description>
<![CDATA[ 科技媒体 borncity ... 失败问题。 ]]>
</description>
<newssource>IT之家</newssource>
<newsauthor>故渊</newsauthor>
<detail>
<![CDATA[ <p data-vmark="65e2">IT之家 10 月 25 日消息， ... （新闻内容略） ...从而引发致命的磁盘读取错误。</p> ]]>
</detail>
<postdate>2025/10/25 9:12:52</postdate>
<hitcount>1421</hitcount>
<commentcount>0</commentcount>
<forbidcomment>false</forbidcomment>
<z>故渊</z>
</item></channel></rss>
```
## 实战
我做这个是因为想拿一些新闻去学校看，下面的python脚本可以自动获取新闻内容并生成一份Microsoft Word(docx)文档用来打印。正文字体为方正书宋，标题为Microsoft Yahei UI Light。<br>
:::info
以下内容由DeepSeek生成
:::
```python
import requests
import json
from datetime import datetime
import time
import random
from docx import Document
from docx.shared import Pt, Inches
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml.ns import qn
import re
import html
import logging
from typing import Optional, Dict, List

# 设置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('ithome_printer.log', encoding='utf-8'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

class ITHomePrinterWithStyles:
    def __init__(self):
        self.base_url = "https://m.ithome.com/api/news/newslistpageget"
        self.news_content_base_url = "https://api.ithome.com/xml/newscontent"
        self.document = Document()
        
        # 重试配置
        self.max_retries = 5
        self.retry_delay = 2
        self.max_delay = 60
        
        # 请求配置
        self.request_timeout = 15
        self.batch_size = 5
        
        # User-Agent轮换
        self.user_agents = [
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36",
            "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36",
            "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36",
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:89.0) Gecko/20100101 Firefox/89.0",
            "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/14.1.1 Safari/605.1.15"
        ]
        
        # 失败队列
        self.failed_news_ids = []
        
        self.setup_styles()
    
    def setup_styles(self):
        """设置文档样式"""
        # 设置正文样式
        style = self.document.styles['Normal']
        font = style.font
        font.name = 'Times New Roman'
        font.size = Pt(9)
        # 设置中文字体
        font.element.rPr.rFonts.set(qn('w:eastAsia'), '方正书宋_GB18030')
        
        # 设置段落格式
        paragraph_format = style.paragraph_format
        paragraph_format.line_spacing = Pt(9)  # 固定值9磅
        paragraph_format.space_before = Pt(0)
        paragraph_format.space_after = Pt(0)
        paragraph_format.widow_control = True  # 孤行控制
        
        # 创建标题样式
        try:
            title_style = self.document.styles.add_style('ITHomeTitle', 1)
        except:
            title_style = self.document.styles['Heading 1']
        
        title_font = title_style.font
        title_font.name = 'Microsoft YaHei UI Light'
        title_font.size = Pt(11)
        title_font.color.rgb = None  # 文字1颜色（黑色）
        # 设置中文字体
        title_font.element.rPr.rFonts.set(qn('w:eastAsia'), 'Microsoft YaHei UI Light')
        
        title_paragraph_format = title_style.paragraph_format
        title_paragraph_format.line_spacing = Pt(13)  # 固定值13磅
        title_paragraph_format.space_before = Pt(0)
        title_paragraph_format.space_after = Pt(0)
        title_paragraph_format.keep_with_next = True  # 与下段同页
        title_paragraph_format.keep_together = True   # 段中不分页
    
    def get_random_user_agent(self):
        """获取随机User-Agent"""
        return random.choice(self.user_agents)
    
    def create_session(self):
        """创建带重试机制的会话"""
        session = requests.Session()
        session.headers.update({
            'User-Agent': self.get_random_user_agent(),
            'Accept': 'application/json, text/plain, */*',
            'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
            'Referer': 'https://m.ithome.com/',
            'Origin': 'https://m.ithome.com'
        })
        return session
    
    def exponential_backoff(self, retry_count):
        """指数退避算法"""
        delay = min(self.retry_delay * (2 ** retry_count), self.max_delay)
        delay += random.uniform(0, 1)
        return delay
    
    def safe_request(self, url, method='GET', params=None, session=None, retry_count=0):
        """安全的请求函数，带重试机制"""
        if retry_count >= self.max_retries:
            logger.error(f"达到最大重试次数: {url}")
            return None
        
        if session is None:
            session = self.create_session()
        
        try:
            if method.upper() == 'GET':
                response = session.get(url, params=params, timeout=self.request_timeout)
            else:
                response = session.get(url, params=params, timeout=self.request_timeout)
            
            if response.status_code == 200:
                logger.info(f"请求成功: {url}")
                return response
            elif response.status_code == 403:
                logger.warning(f"遇到403限制，第{retry_count + 1}次重试: {url}")
                session.headers['User-Agent'] = self.get_random_user_agent()
                delay = self.exponential_backoff(retry_count)
                logger.info(f"等待 {delay:.2f} 秒后重试...")
                time.sleep(delay)
                return self.safe_request(url, method, params, session, retry_count + 1)
            elif response.status_code == 429:
                logger.warning(f"请求过于频繁(429)，第{retry_count + 1}次重试: {url}")
                delay = self.exponential_backoff(retry_count) * 2
                logger.info(f"等待 {delay:.2f} 秒后重试...")
                time.sleep(delay)
                return self.safe_request(url, method, params, session, retry_count + 1)
            else:
                logger.error(f"请求失败，状态码 {response.status_code}: {url}")
                return None
                
        except requests.exceptions.Timeout:
            logger.warning(f"请求超时，第{retry_count + 1}次重试: {url}")
            delay = self.exponential_backoff(retry_count)
            time.sleep(delay)
            return self.safe_request(url, method, params, session, retry_count + 1)
            
        except requests.exceptions.ConnectionError:
            logger.warning(f"连接错误，第{retry_count + 1}次重试: {url}")
            delay = self.exponential_backoff(retry_count)
            time.sleep(delay)
            return self.safe_request(url, method, params, session, retry_count + 1)
            
        except Exception as e:
            logger.error(f"请求异常: {e}, URL: {url}")
            return None
    
    def datetime_to_timestamp(self, dt_str):
        """将日期时间字符串转换为时间戳（毫秒）"""
        try:
            if 'T' in dt_str:
                dt_obj = datetime.fromisoformat(dt_str.replace('Z', '+00:00'))
            else:
                dt_obj = datetime.strptime(dt_str, "%Y-%m-%d %H:%M:%S")
            return int(dt_obj.timestamp() * 1000)
        except Exception as e:
            logger.warning(f"时间转换错误: {e}, 输入: {dt_str}")
            return int(time.time() * 1000)
    
    def get_news_list_page(self, ot=None, page=0):
        """获取单页新闻列表"""
        params = {
            'Tag': '',
            'page': page,
            'hitCountAuthority': 'false'
        }
        
        if ot:
            params['ot'] = ot
            
        url = self.base_url
        response = self.safe_request(url, params=params)
        
        if response:
            try:
                data = response.json()
                if data.get('Success') == 1:
                    news_list = data.get('Result', [])
                    logger.info(f"获取到 {len(news_list)} 条新闻")
                    return news_list
                else:
                    logger.error(f"API返回失败: {data}")
            except json.JSONDecodeError:
                logger.error("JSON解析失败")
        
        return []
    
    def get_multiple_pages(self, total_articles=50, initial_ot=None):
        """获取多页新闻"""
        all_news = []
        current_ot = initial_ot
        page = 0
        
        while len(all_news) < total_articles:
            logger.info(f"获取第 {page + 1} 页新闻...")
            page_news = self.get_news_list_page(current_ot, page)
            
            if not page_news:
                logger.info("没有更多新闻了")
                break
            
            # 过滤广告
            valid_news = []
            for news in page_news:
                is_ad = False
                news_tips = news.get('NewsTips', [])
                for tip in news_tips:
                    if tip.get('TipName') == '广告':
                        is_ad = True
                        break
                
                if not is_ad and news.get('title'):
                    valid_news.append(news)
            
            all_news.extend(valid_news)
            logger.info(f"当前总计: {len(all_news)} 条有效新闻")
            
            # 更新ot参数
            if valid_news:
                last_news = valid_news[-1]
                last_orderdate = last_news.get('orderdate')
                if last_orderdate:
                    current_ot = self.datetime_to_timestamp(last_orderdate)
                    logger.info(f"下一页ot参数: {current_ot}")
            
            page += 1
            time.sleep(1)
            
            if page >= 20:
                logger.warning("达到最大页数限制")
                break
        
        return all_news[:total_articles]
    
    def clean_html(self, text):
        """清理HTML"""
        if not text:
            return ""
        text = html.unescape(text)
        clean = re.compile('<.*?>')
        text = re.sub(clean, '', text)
        text = text.replace('<![CDATA[', '').replace(']]>', '')
        text = re.sub(r'\s+', ' ', text).strip()
        return text
    
    def get_news_content_with_retry(self, newsid):
        """带重试机制的获取新闻内容"""
        newsid_str = str(newsid)
        prefix = newsid_str[:3]
        suffix = newsid_str[3:]
        url = f"{self.news_content_base_url}/{prefix}/{suffix}.xml"
        
        response = self.safe_request(url)
        
        if response:
            try:
                content = response.text
                title_match = re.search(r'<title><!\[CDATA\[(.*?)\]\]></title>', content)
                detail_match = re.search(r'<detail><!\[CDATA\[(.*?)\]\]></detail>', content)
                author_match = re.search(r'<newsauthor>(.*?)</newsauthor>', content)
                postdate_match = re.search(r'<postdate>(.*?)</postdate>', content)
                
                return {
                    'title': self.clean_html(title_match.group(1)) if title_match else "",
                    'content': self.clean_html(detail_match.group(1)) if detail_match else "",
                    'author': self.clean_html(author_match.group(1)) if author_match else "",
                    'postdate': self.clean_html(postdate_match.group(1)) if postdate_match else ""
                }
            except Exception as e:
                logger.error(f"解析内容失败 (ID: {newsid}): {e}")
        
        return None
    
    def process_news_batch(self, news_batch):
        """处理一批新闻"""
        results = []
        for news in news_batch:
            newsid = news.get('newsid')
            title = news.get('title', '')
            
            logger.info(f"处理文章: {title[:50]}...")
            
            content_data = self.get_news_content_with_retry(newsid)
            
            if content_data and content_data.get('content'):
                results.append((news, content_data))
            else:
                self.failed_news_ids.append(newsid)
                logger.warning(f"文章内容获取失败，已加入重试队列: {title[:50]}...")
            
            time.sleep(random.uniform(0.5, 1.5))
        
        return results
    
    def retry_failed_news(self):
        """重试失败的新闻"""
        if not self.failed_news_ids:
            return []
        
        logger.info(f"开始重试 {len(self.failed_news_ids)} 篇失败的文章...")
        
        retry_results = []
        still_failed = []
        
        for newsid in self.failed_news_ids[:]:
            content_data = self.get_news_content_with_retry(newsid)
            
            if content_data and content_data.get('content'):
                retry_results.append(({'newsid': newsid}, content_data))
                self.failed_news_ids.remove(newsid)
                logger.info(f"重试成功: {newsid}")
            else:
                still_failed.append(newsid)
                logger.warning(f"重试仍然失败: {newsid}")
            
            time.sleep(random.uniform(1, 2))
        
        self.failed_news_ids = still_failed
        
        return retry_results
    
    def format_content(self, content):
        """格式化内容"""
        if not content:
            return ""
        
        paragraphs = []
        current_para = []
        
        sentences = [s.strip() for s in content.split('。') if s.strip()]
        
        for i, sentence in enumerate(sentences):
            current_para.append(sentence)
            
            if len(current_para) >= 2 or i == len(sentences) - 1:
                paragraphs.append('。'.join(current_para) + '。')
                current_para = []
        
        return '\n\n'.join(paragraphs)
    
    def add_news_to_document(self, news_data, content_data, is_first_news=False):
        """添加新闻到文档"""
        try:
            title = news_data.get('title', '')
            description = news_data.get('description', '')
            orderdate = news_data.get('orderdate', '')
            
            if content_data and content_data.get('content'):
                full_content = content_data['content']
                author = content_data.get('author', '')
                postdate = content_data.get('postdate', '')
            else:
                full_content = description if description else "暂无详细内容"
                author = ""
                postdate = orderdate
            
            # 如果不是第一条新闻，先添加空行
            if not is_first_news:
                self.document.add_paragraph()
            
            # 添加标题（使用自定义标题样式）
            title_para = self.document.add_paragraph(style='ITHomeTitle')
            title_run = title_para.add_run(title)
            
            # 添加元信息（使用正文样式）
            meta_info = []
            if author:
                meta_info.append(f"作者: {author}")
            if postdate:
                simple_date = postdate.split(' ')[0] if ' ' in postdate else postdate[:10]
                meta_info.append(f"时间: {simple_date}")
            
            if meta_info:
                meta_para = self.document.add_paragraph()
                meta_run = meta_para.add_run(" | ".join(meta_info))
            
            # 添加内容（使用正文样式）
            if full_content:
                formatted_content = self.format_content(full_content)
                for para in formatted_content.split('\n\n'):
                    if para.strip():
                        content_para = self.document.add_paragraph()
                        content_run = content_para.add_run(para.strip())
            
            return True
            
        except Exception as e:
            logger.error(f"添加文章到文档失败: {e}")
            return False
    
    def generate_newspaper(self, total_articles=30, output_filename=None, max_retry_rounds=3):
        """生成报纸"""
        logger.info("🚀 IT之家报纸生成器（标准样式版）启动")
        
        # 创建文档标题（使用正文样式）
        title_para = self.document.add_paragraph()
        title_run = title_para.add_run(f'IT之家报纸 - {datetime.now().strftime("%Y年%m月%d日")}')
        title_run.bold = True
        
        info_para = self.document.add_paragraph()
        info_run = info_para.add_run(f'生成时间: {datetime.now().strftime("%Y-%m-%d %H:%M:%S")}')
        
        # 获取新闻列表
        logger.info("📡 开始获取新闻列表...")
        all_news = self.get_multiple_pages(total_articles)
        
        if not all_news:
            logger.error("❌ 未能获取到任何新闻")
            return False
        
        logger.info(f"✅ 成功收集 {len(all_news)} 条新闻")
        
        # 分批处理新闻
        all_processed_news = []
        batch_count = (len(all_news) + self.batch_size - 1) // self.batch_size
        
        for i in range(batch_count):
            start_idx = i * self.batch_size
            end_idx = min((i + 1) * self.batch_size, len(all_news))
            batch = all_news[start_idx:end_idx]
            
            logger.info(f"处理第 {i + 1}/{batch_count} 批新闻 ({len(batch)} 篇)...")
            
            batch_results = self.process_news_batch(batch)
            all_processed_news.extend(batch_results)
            
           # if i < batch_count - 1:
                #delay = random.uniform(2, 5)
                #logger.info(f"批次间延迟 {delay:.2f} 秒...")
               # time.sleep(delay)
        
        # 重试失败的新闻
        for retry_round in range(max_retry_rounds):
            if not self.failed_news_ids:
                break
                
            logger.info(f"第 {retry_round + 1} 轮重试开始...")
            retry_results = self.retry_failed_news()
            all_processed_news.extend(retry_results)
            
            if self.failed_news_ids:
                logger.warning(f"第 {retry_round + 1} 轮重试后仍有 {len(self.failed_news_ids)} 篇失败")
        
        # 添加到文档
        success_count = 0
        for i, (news, content_data) in enumerate(all_processed_news):
            if self.add_news_to_document(news, content_data, is_first_news=(i == 0)):
                success_count += 1
        
        # 保存文档
        if not output_filename:
            output_filename = f"IT之家报纸_标准样式_{datetime.now().strftime('%Y%m%d_%H%M%S')}.docx"
        
        try:
            self.document.save(output_filename)
            logger.info(f"🎉 报纸生成完成！")
            logger.info(f"📁 文件: {output_filename}")
            logger.info(f"📊 统计: {success_count}/{len(all_processed_news)} 篇文章处理成功")
            
            if self.failed_news_ids:
                logger.warning(f"⚠️ 仍有 {len(self.failed_news_ids)} 篇文章失败: {self.failed_news_ids}")
            
            return True
        except Exception as e:
            logger.error(f"❌ 保存文件失败: {e}")
            return False

def main():
    printer = ITHomePrinterWithStyles()
    
    print("=" * 60)
    print("     IT之家报纸生成器（标准样式版）")
    print("=" * 60)
    print("严格按照规定格式生成:")
    print("  • 标题: Microsoft YaHei UI Light, 11磅, 固定行距13磅")
    print("  • 正文: 方正书宋_GB18030/Times New Roman, 9磅, 固定行距9磅")
    print("  • 段落间距: 无间隔")
    print("  • 新闻间: 空一行")
    print()
    
    try:
        total = input("请输入要生成的新闻总数 (默认30): ").strip()
        total_articles = int(total) if total.isdigit() else 30
        
        retry_rounds = input("请输入最大重试轮次 (默认3): ").strip()
        max_retry_rounds = int(retry_rounds) if retry_rounds.isdigit() else 3
        
        filename = input("请输入输出文件名 (默认自动生成): ").strip()
        filename = filename if filename else None
        
        print("\n开始生成报纸...")
        success = printer.generate_newspaper(total_articles, filename, max_retry_rounds)
        
        if success:
            print("\n✅ 程序执行成功！")
            print("📋 格式说明:")
            print("   - 标题: Microsoft YaHei UI Light, 11磅")
            print("   - 正文: 方正书宋_GB18030/Times New Roman, 9磅")
            print("   - 行距: 标题13磅，正文9磅")
            print("   - 段落: 无间隔，新闻间空一行")
        else:
            print("\n❌ 程序执行失败")
            
    except KeyboardInterrupt:
        print("\n\n⏹️ 用户取消操作")
    except Exception as e:
        print(f"\n❌ 程序执行出错: {e}")

if __name__ == "__main__":
    main()
```
我正在考虑将其上传至GitHub。欢迎各位访问。