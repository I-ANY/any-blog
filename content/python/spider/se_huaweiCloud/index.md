+++
title = "se_huaweiCloud"
date = "2026-05-28T00:01:08+08:00"
draft = false
+++

```python
from selenium.webdriver import Chrome
from selenium.webdriver.common.keys import Keys
from selenium.webdriver import ActionChains
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support import select
from selenium.webdriver.common.by import By      # 匹配标签使用
from selenium.webdriver.chrome.service import Service
import time

class se_huaweicCloud(object):
    def __init__(self):
        self.url = 'https://auth.huaweicloud.com/authui/login.html?service=https://console.huaweicloud.com/marketplace/tenant/#/login'
        self.phone = '17377042972'
        self.passwd = 'weizhen@huawei'
        #实例化浏览器并规避检测selenium
        self.opt = Options()
        self.opt.add_argument('--disable-blink-features=AutomationControlled')
        self.opt.add_experimental_option("excludeSwitches", ['enable-automation', 'enable-logging'])
        #self.brower = Chrome(options=self.opt)
        self.service = Service("D:/Software/python3.7/chromedriver.exe")
        self.brower = Chrome(service=self.service)
        self.brower.implicitly_wait(10)


    #鼠标点击操作
    def Click(self,line):
        action = ActionChains(self.brower)
        action.click_and_hold(line)     #点击不松开
        action.move_by_offset(300,0)    #移动
        action.perform()                #执行动作
        action.release()                #释放鼠标
    #进入到网页
    def Broswer(self):
        #找到账号和密码框输入账号和密码
        self.brower.maximize_window()
        # Selenium在打开任何页面之前，先运行这个Js文件。
        with open('E:/学习资料/我的资料整理/python/spider/hide.js') as f:
            self.brower.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {"source": f.read()})
        self.brower.get(self.url)
        self.brower.find_element(By.XPATH, '//*[@id="hwidsdk_mudule_container"]/div[2]/div/div[2]/div/div/div[1]/div[2]/form/div[2]/div[2]/div/div/div[1]/input').send_keys(self.phone)
        self.brower.find_element(By.XPATH, '//*[@id="hwidsdk_mudule_container"]/div[2]/div/div[2]/div/div/div[1]/div[2]/form/div[3]/div/div/div/input').send_keys(self.passwd)
        self.brower.find_element(By.XPATH,'//*[@id="hwidsdk_mudule_container"]/div[2]/div/div[2]/div/div/div[1]/div[2]/div[2]/div/div/div').click()

        self.brower.find_element(By.XPATH,'//*[@id="hwidsdk_mudule_container"]/div[2]/div/div[2]/div/div/div[1]/div[2]/form/div[3]/div[2]/div[2]/div/div[2]').click()
        self.brower.find_element(By.XPATH, '//*[@id="hwidsdk_mudule_container"]/div[2]/div/div[2]/div/div/div[1]/div[2]/form/div[4]/div/div/div/input').send_keys(self.passwd)
        self.brower.find_element(By.XPATH,'//*[@id="hwidsdk_mudule_container"]/div[2]/div/div[2]/div/div/div[1]/div[2]/div[2]/div/div/div').click()

        # 获取数据
        tag_list =  self.brower.find_elements(By.XPATH, '//*[@id="ti_auto_id_3"]/div/table/tbody/tr[2]')
        for tag in tag_list:
            print(tag.find_element(By.XPATH, 'div/div[2]/div').text)
        # pro_name = self.brower.find_element(By.XPATH, '//*[@id="contentName00"]/div/div[2]/div').text
        #print(pro_name)

        time.sleep(3)

if __name__ == "__main__":
    selenuim = se_huaweicCloud()
    selenuim.Broswer()

```

