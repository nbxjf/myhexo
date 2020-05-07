---
title: python-study
date: 2016-11-13 21:20:19
categories: Python
tags:
 - python 
---

9月28日，我正式开始了我的实习之旅。面试的工作ETL与API开发实习生，但是进入了部门之后，部门中用的很大一部门是python的代码，所以，我开始学习Python大法了
<!--more-->

### python爬虫入门
* 说起python，很直观的感觉就是python的爬虫功能十分强大，打开github搜索python爬虫，数不清的demo，这给我的入门学习提供了很好的案例参考

##### 第一个demo，糗事百科爬虫
* 模仿网上的案例教程，写了第一个爬取糗事百科的crawler
	* urlib2是使用各种协议完成打开url的一个扩展包，最简单的使用方式是调用urlopen方法
	
			def urlopen(url, data=None, timeout=socket._GLOBAL_DEFAULT_TIMEOUT):
	* 可以接受一个字符串型的url地址或者一个Request对象，将打开这个url并返回结果为一个像文件对象一样的对象
* 完成代码如下
	
			```python
			# -*- coding: utf-8 -*-
	
			import urllib2
			import re
			import thread
			import time
	
	
			# ----------- 加载处理糗事百科 -----------
			class Spider_Model:
			    def __init__(self):
			        self.page = 1
			        self.pages = []
			        self.enable = False
			
			    def getPage(self, page):
			        url = "http://m.qiushibaike.com/hot/page/" + str(page)
			        user_agent = 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)'
			        headers = {'User-Agent': user_agent}
			        req = urllib2.Request(url, headers=headers)
			        response = urllib2.urlopen(req)
			        myPage = response.read()
			        unicodePage = myPage.decode("utf-8")
			
			        myItems = re.findall('<div.*?class="content">(.*?)<span>(.*?)</span>(.*?)</div>', unicodePage, re.S)
			        items = []
			        for item in myItems:
			            print item[1]
			            items.append(item[1])
			        return items
			
			    def LoadData(self):
			        while self.enable:
			            if len(self.pages) < 2:
			                try:
			                    myPage = self.getPage(self.page)
			                    self.page += 1
			                    self.pages.append(myPage)
			                except:
			                    print "无法链接糗事百科"
			            else:
			                time.sleep(1)
			
			    def ShowPage(self, nowPage):
			        for item in nowPage:
			            print item
			            # print self.pages
			
			    def Start(self):
			        self.enable = True
			
			        thread.start_new_thread(self.LoadData, ())
			
			        while self.enable:
			            # self.pages数组中有元素
			            if self.pages:
			                nowPage = self.pages[0]
			                del self.pages[0]
			                self.ShowPage(nowPage)
			
			                # ----------- 程序的入口处 -----------
			
			
			print u"""
			---------------------------------------
			   程序：糗百爬虫
			   版本：1.0
			   作者：Jeff_xu
			   日期：2016-10-18
			   语言：Python 2.7
			   操作：输入quit退出阅读糗事百科
			   功能：按下回车依次浏览今日的糗百热点
			---------------------------------------
			"""
			
			print u'请按下回车浏览今日的糗百内容：'
			raw_input(' ')
			myModel = Spider_Model()
			myModel.Start()

##### 第二个demo，爬取百度贴吧
* re模块（正则表达式）
	* re.match(pattern, string, flags)
		* 第一个参数是正则表达式，这里为"(\w+)\s"，如果匹配成功，则返回一个Match，否则返回一个None；
		* 第二个参数表示要匹配的字符串；
		* 第三个参数是标致位，用于控制正则表达式的匹配方式，如：是否区分大小写，多行匹配等等
	*  re.search(pattern, string, flags)
		*  每个参数的含意与re.match一样
		*  re.match与re.search的区别：re.match只匹配字符串的开始，如果字符串开始不符合正则表达式，则匹配失败，函数返回None；而re.search匹配整个字符串，直到找到一个匹配
	* re.sub
	* re.findall
		* re.findall可以获取字符串中所有匹配的字符串
	* re.compile
		* 可以把正则表达式编译成一个正则表达式对象
		
* 详细代码

		```python
		# -*- coding: utf-8 -*-
		
		import string
		import urllib2
		import re
		
		
		# ----------- 处理页面上的各种标签 -----------
		class HTML_Tool:
		    # 用非 贪婪模式 匹配 \t 或者 \n 或者 空格 或者 超链接 或者 图片
		    BgnCharToNoneRex = re.compile("(\t|\n| |<a.*?>|<img.*?>)")
		
		    # 用非 贪婪模式 匹配 任意<>标签
		    EndCharToNoneRex = re.compile("<.*?>")
		
		    # 用非 贪婪模式 匹配 任意<p>标签
		    BgnPartRex = re.compile("<p.*?>")
		    CharToNewLineRex = re.compile("(<br/>|</p>|<tr>|<div>|</div>)")
		    CharToNextTabRex = re.compile("<td>")
		
		    # 将一些html的符号实体转变为原始符号
		    replaceTab = [("<", "<"), (">", ">"), ("&", "&"), ("&", "\""), (" ", " ")]
		
		    def Replace_Char(self, x):
		        x = self.BgnCharToNoneRex.sub("", x)
		        x = self.BgnPartRex.sub("\n    ", x)
		        x = self.CharToNewLineRex.sub("\n", x)
		        x = self.CharToNextTabRex.sub("\t", x)
		        x = self.EndCharToNoneRex.sub("", x)
		
		        for t in self.replaceTab:
		            x = x.replace(t[0], t[1])
		        return x
		
		
		class Baidu_Spider:
		    # 申明相关的属性
		    def __init__(self, url):
		        self.myUrl = url + '?see_lz=1'
		        self.datas = []
		        self.myTool = HTML_Tool()
		        print u'已经启动百度贴吧爬虫，咔嚓咔嚓'
		
		        # 初始化加载页面并将其转码储存
		
		    def baidu_tieba(self):
		        # 读取页面的原始信息并将其从gbk转码
		        myPage = urllib2.urlopen(self.myUrl).read().decode("utf-8")
		        # 计算楼主发布内容一共有多少页
		        endPage = self.page_counter(myPage)
		        # 获取该帖的标题
		        title = self.find_title(myPage)
		        print u'文章名称：' + title
		        # 获取最终的数据
		        self.save_data(self.myUrl, title, endPage)
		
		        # 用来计算一共有多少页
		
		    def page_counter(self, myPage):
		        # 匹配 "共有<span class="red">12</span>页" 来获取一共有多少页
		        myMatch = re.search(r'class="red">(\d+?)</span>', myPage, re.S)
		        if myMatch:
		            endPage = int(myMatch.group(1))
		            print u'爬虫报告：发现楼主共有%d页的原创内容' % endPage
		        else:
		            endPage = 0
		            print u'爬虫报告：无法计算楼主发布内容有多少页！'
		        return endPage
		
		        # 用来寻找该帖的标题
		
		    def find_title(self, myPage):
		        # 匹配 <h1 class="core_title_txt" title="">xxxxxxxxxx</h1> 找出标题
		        myMatch = re.search(r'<h1.*?>(.*?)</h1>', myPage, re.S)
		        title = u'暂无标题'
		        if myMatch:
		            title = myMatch.group(1)
		        else:
		            print u'爬虫报告：无法加载文章标题！'
		            # 文件名不能包含以下字符： \ / ： * ? " < > |
		        title = title.replace('\\', '').replace('/', '').replace(':', '').replace('*', '').replace('?', '').replace('"',
		                                                                                                                    '').replace(
		            '>', '').replace('<', '').replace('|', '')
		        return title
		
		
		        # 用来存储楼主发布的内容
		
		    def save_data(self, url, title, endPage):
		        # 加载页面数据到数组中
		        self.get_data(url, endPage)
		        # 打开本地文件
		        f = open(title + '.txt', 'w+')
		        f.writelines(self.datas)
		        f.close()
		        print u'爬虫报告：文件已下载到本地并打包成txt文件'
		        print u'请按任意键退出...'
		        raw_input();
		
		        # 获取页面源码并将其存储到数组中
		
		    def get_data(self, url, endPage):
		        url = url + '&pn='
		        for i in range(1, endPage + 1):
		            print u'爬虫报告：爬虫%d号正在加载中...' % i
		            myPage = urllib2.urlopen(url + str(i)).read()
		            # 将myPage中的html代码处理并存储到datas里面
		            self.deal_data(myPage.decode('utf-8'))
		
		
		            # 将内容从页面代码中抠出来
		
		    def deal_data(self, myPage):
		        myItems = re.findall('id="post_content.*?>(.*?)</div>', myPage, re.S)
		        for item in myItems:
		            data = self.myTool.Replace_Char(item.replace("\n", "").encode('utf-8'))
		            self.datas.append(data + '\n')
		
		
		
		            # -------- 程序入口处 ------------------
		
		
		print u"""#---------------------------------------
		#   程序：百度贴吧爬虫
		#   版本：1.0
		#   作者：Jeff
		#   日期：2016-10-18
		#   语言：Python 2.7
		#   操作：输入网址后自动只看楼主并保存到本地文件
		#   功能：将楼主发布的内容打包txt存储到本地。
		#---------------------------------------
		"""
		
		print u'请输入贴吧的地址最后的数字串：'
		bdurl = 'http://tieba.baidu.com/p/' + str(raw_input(u'http://tieba.baidu.com/p/'))
		
		# 调用
		mySpider = Baidu_Spider(bdurl)
		mySpider.baidu_tieba()
		```

