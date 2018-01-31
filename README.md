## 项目依赖

#### Ubuntu

1.安装requests,chardet,web.py,gevent psutil:
pip install requests chardet web.py sqlalchemy gevent psutil
<br/>
2.安装lxml:
apt-get install python-lxml
sudo apt-get install libffi-dev libssl-dev

<br/>
注意：


本项目默认数据库是sqlite，但是采用sqlalchemy的ORM模型，通过预留接口可以拓展使用MySQL，MongoDB等数据库。
配置方法：
<br/>
1.MySQL配置
```
第一步：首先安装MySQL数据库并启动
第二步：安装MySQLdb或者pymysql(推荐)
第三步：在config.py文件中配置DB_CONFIG。如果安装的是MySQLdb模块，配置如下：
        DB_CONFIG={
            'DB_CONNECT_TYPE':'sqlalchemy',
            'DB_CONNECT_STRING':'mysql+mysqldb://root:root@localhost/proxy?charset=utf8'
        }
        如果安装的是pymysql模块，配置如下：
         DB_CONFIG={
            'DB_CONNECT_TYPE':'sqlalchemy',
            'DB_CONNECT_STRING':'mysql+pymysql://root:root@localhost/proxy?charset=utf8'
        }
```
sqlalchemy下的DB_CONNECT_STRING参考[支持数据库](http://docs.sqlalchemy.org/en/latest/core/engines.html#supported-databases)，理论上使用这种配置方式不只是适配MySQL，sqlalchemy支持的数据库都可以，但是仅仅测试过MySQL。
<br/>
2.MongoDB配置
```
第一步：首先安装MongoDB数据库并启动
第二步：安装pymongo模块
第三步：在config.py文件中配置DB_CONFIG。配置类似如下：
        DB_CONFIG={
            'DB_CONNECT_TYPE':'pymongo',
            'DB_CONNECT_STRING':'mongodb://localhost:27017/'
        }
```
由于sqlalchemy并不支持MongoDB,因此额外添加了pymongo模式，DB_CONNECT_STRING参考pymongo的连接字符串。

##### 注意

如果大家想拓展其他数据库，可以直接继承db下ISqlHelper类，实现其中的方法，具体实现参考我的代码，然后在DataStore中导入类即可。
```
try:
    if DB_CONFIG['DB_CONNECT_TYPE'] == 'pymongo':
        from db.MongoHelper import MongoHelper as SqlHelper
    else:
        from db.SqlHelper import SqlHelper as SqlHelper
    sqlhelper = SqlHelper()
    sqlhelper.init_db()
except Exception,e:
    raise Con_DB_Fail
```
有感兴趣的朋友，可以将Redis的实现方式添加进来。


## 如何使用

将项目目录clone到当前文件夹

$ git clone 

切换工程目录

```
$ cd IPProxyPool
```

运行脚本

```
python IPProxy.py
```
成功运行后，打印信息
```
IPProxyPool----->>>>>>>>beginning
http://0.0.0.0:8000/
IPProxyPool----->>>>>>>>db exists ip:0
IPProxyPool----->>>>>>>>now ip num < MINNUM,start crawling...
IPProxyPool----->>>>>>>>Success ip num :134,Fail ip num:7882
```

## API 使用方法

#### 第一种模式
```
GET /
```
这种模式用于查询代理ip数据，同时加入评分机制，返回数据的顺序是按照评分由高到低，速度由快到慢制定的。

#### 参数 

| Name | Type | Description |
| ----| ---- | ---- |
| types | int | 0: 高匿,1:匿名,2 透明 |
| protocol | int | 0: http, 1 https, 2 http/https |
| count | int | 数量 |
| country | str | 取值为 国内, 国外 |
| area | str | 地区 |



#### 例子

##### IPProxys默认端口为8000,端口可以在config.py中配置。

##### 如果是在本机上测试：

1.获取5个ip地址在中国的高匿代理：http://127.0.0.1:8000/?types=0&count=5&country=国内
<br/>
2.响应为JSON格式，按照评分由高到低，响应速度由高到低的顺序，返回数据：
<br/>
```
[["122.226.189.55", 138, 10], ["183.61.236.54", 3128, 10], ["61.132.241.109", 808, 10], ["183.61.236.53", 3128, 10], ["122.227.246.102", 808, 10]]
```
<br/>
以["122.226.189.55", 138, 10]为例，第一个元素是ip,第二个元素是port，第三个元素是分值score。

```
import requests
import json
r = requests.get('http://127.0.0.1:8000/?types=0&count=5&country=国内')
ip_ports = json.loads(r.text)
print ip_ports
ip = ip_ports[0][0]
port = ip_ports[0][1]
proxies={
    'http':'http://%s:%s'%(ip,port),
    'https':'http://%s:%s'%(ip,port)
}
r = requests.get('http://ip.chinaz.com/',proxies=proxies)
r.encoding='utf-8'
print r.text
```
#### 第二种模式
```
GET /delete
```
这种模式用于方便用户根据自己的需求删除代理ip数据

#### 参数 

| Name | Type | Description |
| ----| ---- | ---- |
| ip | str | 类似192.168.1.1 |
| port | int | 类似 80 |
| types | int |  0: 高匿,1:匿名,2 透明 |
| protocol | int | 0: http, 1 https, 2 http/https |
| count | int | 数量 |
| country | str | 取值为 国内, 国外 |
| area | str | 地区 |

大家可以根据指定以上一种或几种方式删除数据。

#### 例子

##### 如果是在本机上测试：

1.删除ip为120.92.3.127的代理：http://127.0.0.1:8000/delete?ip=120.92.3.127
<br/>
2.响应为JSON格式，返回删除的结果为成功,失败或者返回删除的个数,类似如下的效果：
["deleteNum", "ok"]或者["deleteNum", 1]
```
import requests
r = requests.get('http://127.0.0.1:8000/delete?ip=120.92.3.127')
print r.text
```
## config.py参数配置
```
#parserList是网址解析规则表,大家可以将发现的代理网址,将提取规则添加到其中,方便爬虫的爬取。
parserList = [
    {
        'urls': ['http://www.66ip.cn/%s.html' % n for n in ['index'] + list(range(2, 12))],
        'type': 'xpath',
        'pattern': ".//*[@id='main']/div/div[1]/table/tr[position()>1]",
        'position': {'ip': './td[1]', 'port': './td[2]', 'type': './td[4]', 'protocol': ''}
    },
    
   ......
 
   
    {
        'urls': ['http://www.cnproxy.com/proxy%s.html' % i for i in range(1, 11)],
        'type': 'module',
        'moduleName': 'CnproxyPraser',
        'pattern': r'<tr><td>(\d+\.\d+\.\d+\.\d+)<SCRIPT type=text/javascript>document.write\(\"\:\"(.+)\)</SCRIPT></td><td>(HTTP|SOCKS4)\s*',
        'position': {'ip': 0, 'port': 1, 'type': -1, 'protocol': 2}
    }
]

#数据库的配置

DB_CONFIG = {

    'DB_CONNECT_TYPE': 'sqlalchemy',  # 'pymongo'sqlalchemy;redis
    # 'DB_CONNECT_STRING':'mongodb://localhost:27017/'
    'DB_CONNECT_STRING': 'sqlite:///' + os.path.dirname(__file__) + '/data/proxy.db'
    # DB_CONNECT_STRING : 'mysql+mysqldb://root:root@localhost/proxy?charset=utf8'

    # 'DB_CONNECT_TYPE': 'redis',  # 'pymongo'sqlalchemy;redis
    # 'DB_CONNECT_STRING': 'redis://localhost:6379/8',

}
#THREADNUM为gevent pool的协程数目
THREADNUM = 5

#API_PORT为API web服务器的端口
API_PORT = 8000

#爬虫爬取和检测ip的设置条件
#不需要检测ip是否已经存在，因为会定时清理
# UPDATE_TIME:每半个小时检测一次是否有代理ip失效
UPDATE_TIME = 30 * 60 

# 当有效的ip值小于MINNUM时 需要启动爬虫进行爬取
MINNUM = 50  

# socket超时
TIMEOUT = 5 




#爬虫下载网页的重试次数
RETRY_TIME = 3


#USER_AGENTS 随机头信息,用来突破爬取网站的反爬虫

USER_AGENTS = [
    "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; AcooBrowser; .NET CLR 1.1.4322; .NET CLR 2.0.50727)",
    "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.0; Acoo Browser; SLCC1; .NET CLR 2.0.50727; Media Center PC 5.0; .NET CLR 3.0.04506)",
    "Mozilla/4.0 (compatible; MSIE 7.0; AOL 9.5; AOLBuild 4337.35; Windows NT 5.1; .NET CLR 1.1.4322; .NET CLR 2.0.50727)",
    "Mozilla/5.0 (Windows; U; MSIE 9.0; Windows NT 9.0; en-US)",
    "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET CLR 2.0.50727; Media Center PC 6.0)",
    "Mozilla/5.0 (compatible; MSIE 8.0; Windows NT 6.0; Trident/4.0; WOW64; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET CLR 1.0.3705; .NET CLR 1.1.4322)",
    "Mozilla/4.0 (compatible; MSIE 7.0b; Windows NT 5.2; .NET CLR 1.1.4322; .NET CLR 2.0.50727; InfoPath.2; .NET CLR 3.0.04506.30)",
   ]
#默认给抓取的ip分配10分,每次连接失败,减一分,直到分数全部扣完从数据库中删除
DEFAULT_SCORE=10

