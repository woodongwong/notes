![CGI Logo](https://github.com/woodongwong/Blog/blob/master/Images/CGILogo.gif?raw=true)

### 1、什么是CGI？

> 通用网关接口（Common Gateway Interface/CGI）是一种重要的互联网技术，可以让一个客户端，从网页浏览器向执行在网络服务器上的程序请求数据。CGI描述了服务器和请求处理程序之间传输数据的一种标准。最初，CGI是在1993年由美国国家超级电脑应用中心（NCSA）为NCSA HTTPd Web服务器开发的。这个Web服务器使用了UNIX shell 环境变量来保存从Web服务器传递出去的参数，然后生成一个运行CGI的独立的进程。CGI程序可以用任何脚本语言或者是完全独立编程语言实现，只要这个语言可以在这个系统上运行。像Unix shell script, Python, Ruby, PHP, Tcl, C/C++,和Visual Basic都可以用来编写CGI程序。（摘自[维基百科](https://zh.wikipedia.org/wiki/%E9%80%9A%E7%94%A8%E7%BD%91%E5%85%B3%E6%8E%A5%E5%8F%A3)）

---

### 2、什么是CGI程序？

在第1段中所提到的“请求处理程序”就是CGI程序。

既然这么多编程语言都可以编写CGI程序，我们不妨也来试试，举个栗子：（假设你对linux和web环境有一定的了解，那我们就用bash shell来编写CGI程序）

```
# !/bin/sh
echo -e "Content-Type:text/html\n\n"
echo "hello CGI"
```
echo -e 参数是处理特殊字符，会把"\n"以换行符进行输出。

问题：为什么要输出"Content-Type:text/html\n\n"？<br>答：在[RFC3875](https://tools.ietf.org/html/rfc3875)中规定:"script MUST supply a Content-Type field in the response" 脚本响应时必须提供Content-Type字段。"\n\n"是分割HTTP协议中header与body的。

---

### 3、CGI程序是如何运行的？

每一次请求都会生成一个运行CGI的独立的进程，既然是独立进程，使用linux中的"ps"命令就可以查看得到，我们来做一个实验，"逮住"这个进程。

```
# !/bin/sh
sleep 20
echo -e "Content-Type:text/html\n\n"
echo "hello CGI"
```
"sleep 20"："睡眠"20秒，方便我们查看CGI程序的进程

```
# ps -ef | grep index.sh
UID        PID  PPID  C STIME TTY          TIME CMD
www      19848 13642  0 21:55 ?        00:00:00 /bin/sh /opt/web/cgi-bin/index.cgi
```
```
# ps -ef | grep httpd
UID        PID  PPID  C STIME TTY          TIME CMD
root     13598     1  0 Jul30 ?        00:00:16 /usr/local/apache2/bin/httpd -f /usr/local/apache2/conf/httpd.conf
www      13642 13598  0 21:53 ?        00:00:00 /usr/local/apache2/bin/httpd -f /usr/local/apache2/conf/httpd.conf
```
有一个很"巧合"的现象：CGI程序进程的父ID(PPID)与Apache的子进程(这里指的worker进程)ID(PID)一样。
其实这个CGI程序进程是由Apache的子进程fork出来的，然后使用了[exec系列函数](https://en.wikipedia.org/wiki/Exec_(system_call))执行的CGI程序，所以CGI程序需要x(执行)权限。

补充一下：测试用的Apache是2.4版本，[多处理模块(MPM)](https://httpd.apache.org/docs/2.4/mpm.html)选用的[worker](https://httpd.apache.org/docs/2.4/mod/worker.html)，可能Apache使用了一些其他技术(比如线程技术)来完成以上工作，大致上原理是这样的。例如 [Boa Webserver](https://en.wikipedia.org/wiki/Boa_(web_server)) fork出来一个子进程后，使用了execve函数执行的CGI程序，并且将上一个程序的环境变量传入。

---

### 4、环境变量

CGI程序是通过临时环境变量获取HTTP详情（在第1段中也提到过）。在web server执行CGI程序时，将环境变量传入或者继承上一个程序的，在[CGI1.1](https://tools.ietf.org/html/rfc3875)标准中,环境变量如下：

| 变量名            | 描述 (以下描述均摘自互联网，仅供参考，英文好的同学可以直接阅读RFC3875)                                        |
| ----------------- | ------------------------------------------------------------------------------------------------------------- |
| AUTH_TYPE         | 服务器验证用户身份所使用的机制                                                                                |
| CONTENT_LENGTH    | 假如REQUEST_METHOD为POST，这个环境变量的值是标准输入中可读取到的有效数据的字节数                              |
| CONTENT_TYPE      | 所传递来的信息的MIME类型                                                                                      |
| GATEWAY_INTERFACE | 服务器遵守的CGI版本                                                                                           |
| PATH_INFO         | CGI程序的附加路径                                                                                             |
| PATH_TRANSLATED   | PATH_INFO对应的绝对路径                                                                                       |
| QUERY_STRING      | GET请求参数                                                                                                   |
| REMOTE_ADDR       | 发送请求的客户机的IP地址                                                                                      |
| REMOTE_HOST       | 递交脚本的主机名                                                                                              |
| REMOTE_IDENT      | 如果Web服务器是在ident (一种确认用户连接你的协议)运行, 递交表单的系统也在运行ident, 这个变量就含有ident返回值 |
| REMOTE_USER       | 递交脚本的用户名. 如果服务器的Authentication被激活，这个值可以设置。                                          |
| REQUEST_METHOD    | 请求方式：GET、POST                                                                                           |
| SCRIPT_NAME       | 指向CGI脚本的路径                                                                                             |
| SERVER_NAME       | 服务器的IP或名字                                                                                              |
| SERVER_PORT       | 主机的端口号                                                                                                  |
| SERVER_PROTOCOL   | 服务器运行的HTTP协议                                                                                          |
| SERVER_SOFTWARE   | 服务器软件的名字                                                                                              |


写个程序验证一下：
```
# !/bin/sh
echo -e "Content-Type:text/html\n\n"
echo $GATEWAY_INTERFACE
```
请求，输出结果为： CGI/1.1

---

### 5、CGI程序如何获取GET和POST请求的数据

* GET

    当GET请求时，REQUEST_METHOD 为 GET；QUERY_STRING 为请求的参数和值，例如：

    http://www.example.org/cgi-bin/demo.cgi/?name=cgi&version=1.1<br>QUERY_STRING 值为 name=cgi&version=1.1

* POST

    POST数据不会存到QUERY_STRING环境变量中，而是从[标准输入(STDIN)](https://zh.wikipedia.org/wiki/%E6%A8%99%E6%BA%96%E4%B8%B2%E6%B5%81#.E6.A8.99.E6.BA.96.E8.BC.B8.E5.85.A5_.28stdin.29)中获取数据，
	具体是如何将请求数据重导向到标准输入(STDIN)的，这涉及到了管道，大概是使用了dup2函数，由于知识有限，本文不再做详解。
    
    以下是读取POST数据的示例
    
    bash shell
    ```
    #!/bin/sh
    echo -e "Content-Type:text/html\n\n"
    echo $(</dev/stdin)
    ```
    php
    ```
    #!/usr/local/php5.6/bin/php
    <?php
    echo "Content-Type:text/html\n\n";
    readfile('php://stdin');
    ?>
    ```

---

### 6、整体请求过程

![request CGI](https://github.com/woodongwong/Blog/blob/master/Images/RequestCGI.png?raw=true)

注：图中数据返回是通过[标准输出(STDOUT)](https://zh.wikipedia.org/wiki/%E6%A8%99%E6%BA%96%E4%B8%B2%E6%B5%81#.E6.A8.99.E6.BA.96.E8.BC.B8.E5.87.BA_.28stdout.29)，然后将数据重导向。具体实现涉及到了管道和dup2函数，本文不做详解。

---

___谢谢阅读！ 如有错误或者不好的地方麻烦指出。共同学习，共同进步~___

（完）

