# tech-log


最近的工作是要使两个系统通过IPv6相连。具体是一个系统能够通过RESTful API连接另外一个系统，只不过要通过IPv6协议。
我只会一点Python的皮毛, 对于TCP/IP/SSL/IPv6了解的也不多。但整个过程下来，确实学了很多东西，在这里记下了，整理一下思路和经验。 

## 问题关键字和问题表象
* 关键字： IPv6, HTTPS, Apache Web Server, Python requests
* 问题描述：使用requests用IPv6访问HTTPS服务的时候，出现400 Bad Request. HTTPS服务是由Apache Server暴露的。
* 类似如下代码：
```python
import ssl
import requests
from requests.auth import HTTPBasicAuth

ip = '[fd99:f17b:37d0::100]'

s = requests.Session()
resp = s.get('https://%s/api/user' % ip, auth=HTTPBasicAuth('admin', 'welcome'), headers=headers, verify=False)
print(resp.content)
```

## 解决方案:

* Python 2.7.12 + Requests 2.10.0 + `ssl.HAS_SNI=False` (可以解决问题)
* Python 3.5.2 + Requests 2.9.1 + `ssl.HAS_SNI=False` (可以解决问题)
* Python 2.7.12 + Requests 2.18.4 + `ssl.HAS_SNI=False` (不可以解决问题)
* Python 2.7.12 + Requests 2.9.1 + `ssl.HAS_SNI=False` (不可以解决问题)


## 都是谁有问题

* Apache有bug。在处理基于IPv6的HTTPS请求的时候，会出现SSL的SNI和HTTP的Host Header不一致的情况。原因是Apache在解析IPv6 Host Header的时候出现了逻辑错误。会把IPv6地址`[fe80::20c:29ff:fee7:e908]`解析成`fe80::20c:29ff:fee7`。怀疑Apache把`:e908`当成了端口号。

* Python的一点问题。Python的_ssl模块的HAS\_SNI属性，应该设置成只读的。它代表的平台上的SSL是否支持SNI，因而不应该让用户修改，以为可以控制HAS\_SNI. 你修改HAS\_SNI是会改变urllib3的行为的，如果设置`ssl.HAS\_SNI=False`, 你确实可以让urllib3不在SSL handshake的时候发送SNI。HAS\_SNI其实代表了openssl库的宏。而宏是编译器的行为，为什么对应了一个可以改变的值，这就是设计的问题。 这个问题困扰了我大概两天的时间。

* Linux的锅。 用工具ping6，在不指定接口的情况下，是ping不同Link Local类型的IPv6地址。其他平台，譬如Windows, Solaris, 是可以直接ping通的。

## 几点感悟
* 新的东西和技术，到处是坑. 例子是IPv6. 平台，工具可能都有问题。

* Python2, Python3。语言的两个版本，会消耗更多的精力和成本，去维护Python的程序。在Python3上运行良好，但是在Python2上运行可能就有问题。

* Python的好处是你可以直接修改语言自带库的代码，测试运行结果。 
* requests是Python的一个库。它提供了更易用的API, 但代价是多一个层次的逻辑和复杂度，灵活性就低了。

* 当出现问题时，你会尝试各种可能，通过实验观察软件运行现象，推测运行逻辑。这个时候，你要准确记录测试结果，各种参数或环境下，出现的各种结果。如果记录错误或者记下的是假象，你的方向就会错误，浪费精力。你可以参考单元测试的方式，按测试用例的方式分门别类的组织(组织的可能是脚本)每个实验，并按表格的方式记录下来。有的时候，你可以从表格中，发现一些规律，甚至可以定位问题的所在模块。

* 要注重错误日志的查看，不要总是想着从代码逻辑去定位问题，代码逻辑万万千千，你的时间却有限，从问题出现的位置开始查起，是最高效的方式。譬如说，你对着Apache发了HTTP请求，但返回是400(Bad Request), 你要看的就是请求有没有到达Apache (access_log), Apache 为什么认为发送过了的请求是错误的(想必错误日志肯定会告诉你原因err\_log)

## 工具与方法：
* tcpdump+wireshark: 确定请求中有没有发送SNI.
    * 收集网络数据包: `sudo tcpdump -X -S -s 0 -i eth0 -vv ip6 -w wireshark.log`
    * 用Wireshark打开上一步收集到的数据。
* Python virtualenv。 用来测试库的不同版本的行为。
* `pip install requests=randomchar`用来查找一个库的所有版本。

## 学到的知识
* SSL/TLS协议与OpenSSL
    * 非对称加密, 私钥。

* IPv6:
    * Scope Local类型的地址
    * Scope Global类型的地址

