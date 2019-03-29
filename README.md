# 批量下载QQ邮箱附件，下载完后修改文件重命名

因为工作原因，需要处理QQ邮箱上来自各地网友的投稿附件。数量比较多，如果手动一个一个下载很麻烦，而且有些发来的附件命名也不规范，下载下来之后还需要手动去重命名，否则放一起就分不清谁是谁了。这种非常机械化的重复操作，我想写个脚本批量下载QQ邮箱附件。

于是临时研究了一下 Python + selenium + Chrome 来节省很多时间~

---

## 如何安装

- **Python 3** https://www.python.org/
- **WebDriver for Chrome** https://sites.google.com/a/chromium.org/chromedriver/downloads

---

## 如何使用

### 1. 运行下面这段代码，手动输入账号密码登陆邮箱,并勾选下次自动登陆
``` python
# encoding:utf-8
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument("user-data-dir=selenium")
chrome = webdriver.Chrome(options=options)

url = 'https://mail.qq.com/'
chrome.get(url)

print('登陆成功！')
```

第一次启动脚本会进入QQ邮箱的登陆页面，需要手动登陆一次。
勾选下次自动登陆，然后就可以关掉浏览器了。下一次再启动脚本，就会自动登陆直接进入邮箱主页了。

> 注：通过运行脚本启动的浏览器窗口，同时只能启动一个。若重复启动脚本将会打开空页面，需要关闭上一个窗口重新运行脚本。


然后把你想要下载的邮件移动到文件夹里面，让脚本进入文件夹里面自动下载。
如果邮件数量比较多，建议在邮箱设置-常规里面，调整每页显示100封邮件。


### 2. 运行`Download_QQEmail_File.py` 

``` python
# encoding:utf-8
import os
import urllib
import _thread
import time
from selenium import webdriver

#......................................................
# 自定义参数
#......................................................
# 登陆邮箱之后，找到你想打开的文件夹，右键新窗口打开，在浏览器地址栏可以看到。
# https://mail.qq.com/cgi-bin/frame_html?t=frame_html&sid={ A }&url=/cgi-bin/mail_list?folderid={ B }%26page={ C }
#
# usid: 相当于临时身份认证吧。（这串密钥似乎会每天更新）
# ufolderid: 文件夹的ID
# upage: 页数 （从0开始计数）
#......................................................
usid,ufolderid,upage = 'R6-CnEyUYkXOuAy','135',0

# 附件下载到哪个文件夹（先提前建好，不然会报错）
download_here = 'D:\\Downloads\\email'

# index:   当前处理的邮件
# istart:  从第几封开始下载
# iend:    下载多少封
index,istart,iend=0,1,-1

# 是否开启Debug模式，只列出邮件列表，不下载任何东西(用来测试用的)
debug = 0

#......................................................
# 配置Web Driver
#......................................................
prefs = {"download.default_directory" : download_here}
options = webdriver.ChromeOptions()
options.add_argument("user-data-dir=selenium")
options.add_experimental_option("prefs",prefs)
chrome = webdriver.Chrome(options=options)

url_qqmail = 'https://mail.qq.com/'
url_folder = url_qqmail+'cgi-bin/mail_list?sid={}&folderid={}&page={}'.format(usid,ufolderid,upage)

# 启动Web Driver
chrome.get(url_qqmail)
print("Chrome启动")

# 进入文件夹
time.sleep(0.5)
chrome.get(url_folder)
time.sleep(0.5)
chrome.switch_to.frame(chrome.find_element_by_id("mainFrame"))
print("---- 进入文件夹: ",chrome.find_element_by_xpath('//*[@id="qqmail_mailcontainer"]/div[1]').text.strip('管理"我的文件夹"').strip())

#......................................................
# 用来测试的一些参数
#......................................................

# readmail: 帖子列表：包含时间戳(totime)、发件人(fn)、发件人邮箱(fa)、邮件id(value)
# filemail: 附件列表：包含附件名、发件人、发件人邮箱、邮件主题
readmail,filemail = [],[]

# 用来计数用的，有时候总感觉会漏掉一些文件。
fileindex,filecount,downloadtimes=0,0,0

#......................................................
# 开始处理邮件
#......................................................

elements = chrome.find_elements_by_css_selector('input[name="mailid"]')
for e in elements:
  if iend == -1:
    iend = len(elements)
  if index > istart:
    if index <= iend+1:
      sender = {}
      sender.update({'timestamp': e.get_attribute('totime')})
      sender.update({'name': e.get_attribute('fn')})
      sender.update({'email': e.get_attribute('fa')})
      sender.update({'id': e.get_attribute('value')})
      readmail.append(sender)
      time.sleep(0.2)
      if debug == 1:
       print('{} {}'.format(index,sender))
  index+=1

print('邮件主题({})'.format(index-1))

if debug != 1:
  for key in readmail:
    url = url_qqmail+'cgi-bin/frame_html?sid={}&url=/cgi-bin/readmail?mailid={}'.format(usid,key['id'])
    url = urllib.parse.unquote(url)
    chrome.get(url)
    chrome.switch_to.default_content()
    chrome.switch_to.frame(chrome.find_element_by_id("mainFrame"))
    chrome.implicitly_wait(5)
    time.sleep(0.5)

    elements = chrome.find_elements_by_class_name("name_big")
    filecount=0

    for f in elements:
      attach={}
      attach.update({'title': chrome.find_element_by_id("subject").text})
      attach.update({'name': key['name']})
      attach.update({'email': key['email']})
      attach.update({'filename': f.find_element_by_css_selector('span:nth-child(1)').text})
      filemail.append(attach)
      filecount+=1
      print("get:",attach)
      time.sleep(1)

    elements = chrome.find_elements_by_link_text('下载')

    for e in elements:
      e.click()
      downloadtimes+=1
      time.sleep(2)
      chrome.implicitly_wait(5)

    # 创建批处理脚本
    os.chdir(download_here)
    cmd = open("_ren.bat","a")
    cmd.seek(0)
    cmd.truncate()
    cmd.write("@echo off")

    for key in filemail:
      cmd.write('ren "{}" "{}-{}"'.format(key['filename'], key['email'], key['filename']))
      cmd.write("\n")

    fileindex+=1
    print('fileindex({}),  filecount({}),  downloadtimes({}) '.format(fileindex,filecount,downloadtimes))
    print("----")

  cmd.write("del _ren.bat")
  cmd.close()

print("----")
print("Done.")

# print('readmail:',readmail)
# print('filemail:',filemail)
```

---

## 踩坑历史
1. 附件收藏中的"全部附件"，并不是想象中真的把全部附件整合在一起，偶尔还是会漏掉一些。
 
 
 
 
