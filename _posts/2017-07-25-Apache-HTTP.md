---
layout: post
title: '基于Apache搭建HTTP/HTTPS/正向代理/反向代理服务器'
date: '2017-07-25'
header-img: "img/post-bg-apache.png"
tags:
     - Apache
author: 'boborz'
---

实际环境
----
+ 系统环境 `macOS Sierra(10.12.5)`
+ Apache `Apache/2.4.25 (Unix)`
+ OpenSSL `OpenSSL 1.0.2l`

引言
----
**Mac**系统默认安装了**Apache**服务器，你只需要在终端里输入`sudo apachectl start`命令，然后打开浏览器，输入网址`http://localhost/index`，显示如下图：

![](http://upload-images.jianshu.io/upload_images/3145770-da86a1e54aa30818.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

恭喜你，一个**HTTP**服务器已经搭建成功了！是不是很简单？接下来介绍如何具体定制化配置**Apache**服务器。

搭建HTTP服务器
----
* **修改服务器默认根路径**   
打开配置文件`/private/etc/apache2/httpd.conf`，更改系统默认的根路径`DocumentRoot `为自定义路径（因为系统默认的根路径要求管理员权限，更改比较繁琐，如果需要用系统默认的根路径，可以跳过此步骤）。

 ```
 DocumentRoot "/Users/libo/apache_server"
 <Directory "/Users/libo/apache_server">
 ```
* **设置虚拟主机**   
通过设置多个虚拟主机可以支持一台物理服务器访问多个域名，就好像有多个服务器一样。打开配置文件`/private/etc/apache2/httpd.conf`，去掉`#Include /private/etc/apache2/extra/httpd-vhosts.conf`前面的`#`，保存并退出。然后打开`/private/etc/apache2/extra/httpd-vhosts.conf`，注释掉以下代码：

 ```
 #<VirtualHost *:80>
 #    ServerAdmin webmaster@dummy-host.example.com
 #    DocumentRoot "/usr/docs/dummy-host.example.com"
 #    ServerName dummy-host.example.com
 #    ServerAlias www.dummy-host.example.com
 #    ErrorLog "/private/var/log/apache2/dummy-host.example.com-error_log"
 #    CustomLog "/private/var/log/apache2/dummy-host.example.com-access_log" common
 #</VirtualHost>

 #<VirtualHost *:80>
 #    ServerAdmin webmaster@dummy-host2.example.com
 #    DocumentRoot "/usr/docs/dummy-host2.example.com"
 #    ServerName dummy-host2.example.com
 #    ErrorLog "/private/var/log/apache2/dummy-host2.example.com-error_log"
 #    CustomLog "/private/var/log/apache2/dummy-host2.example.com-access_log" common
 #</VirtualHost>
 ```
然后加入以下代码：

 ```
 <VirtualHost *:80>
     # ServerAdmin webmaster@dummy-host.example.com
     DocumentRoot "/Users/libo/apache_server/mywebsite"
     ServerName mywebsite.com
     ErrorLog "/private/var/log/apache2/mywebsite.com-error_log"
     CustomLog "/private/var/log/apache2/mywebsite.com-access_log" common

     <Directory />
                 Options Indexes FollowSymLinks MultiViews
                 AllowOverride all
                 Order deny,allow
                 Allow from all
     </Directory>

 </VirtualHost>

 <VirtualHost *:80>
     # ServerAdmin webmaster@dummy-host2.example.com
     DocumentRoot "/Users/libo/apache_server/mywebsite2"
     ServerName mywebsite2.com
     ErrorLog "/private/var/log/apache2/mywebsite2.com-error_log"
     CustomLog "/private/var/log/apache2/mywebsite2.com-access_log" common

     <Directory />
                 Options Indexes FollowSymLinks MultiViews
                 AllowOverride all
                 Order deny,allow
                 Allow from all
     </Directory>
    
 </VirtualHost>
 ```
我们也配置两个虚拟主机，因为**HTTP**不显式指定访问端口号时默认为**80**号端口，为了访问方便，我们设置服务器访问端口号为**80**，当然你也可以设置其它端口号，比如**8080**，这时还需要配置`httpd.conf`，加入`Listen 8080`。但有些端口有特殊意义，不能随便设置，比如**443**端口号默认是HTTPS访问端口。
`<Directory /></Directory>`之间的配置是一些比较细化的服务器配置选项，`AllowOverride all`是允许覆盖所有的文件，`Order deny,allow`是命令的两种类型，`Allow from all`是允许所有的客户端（或代理）访问本服务器，你也可以配置
`Allow from #IP`或`Deny from #IP`来控制允许或者禁止特定的**IP**访问服务器。  

 **注意：两个虚拟主机的根路径必须在服务器根路径下。** 

* **更改本地DNS配置文件**  
打开配置文件`/private/etc/hosts`，把我们的服务器域名对应到回送地址`127.0.0.1`上，这样就可以直接本地进行域名访问。文件前3行配置是系统默认配置，我们平时访问的`localhost`就是通过该文件直接解析成对应的**IP**地址进行本地访问的，其中`127.0.0.1`是`IPv4`标准，`::1`是`IPv6`标准。其实我们平时通过域名访问网址时，都会首先访问这个本地的`hosts`文件来查找**IP**地址，如果找不到对应的结果才会访问DNS服务器进行DNS解析来进行后续的步骤。  
配置代码如下：

 ```
 ##
 # Host Database
 #
 # localhost is used to configure the loopback interface
 # when the system is booting.  Do not change this entry.
 ##
 127.0.0.1  localhost
 255.255.255.255  broadcasthost
 ::1             localhost
 127.0.0.1       mywebsite.com
 127.0.0.1       mywebsite2.com
 ```
* **验证**   
最后我们在配置的服务器根路径下加入测试的**HTML**文件来进行测试，结果如下：

![](http://upload-images.jianshu.io/upload_images/3145770-4fa406fc2291e22e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

![](http://upload-images.jianshu.io/upload_images/3145770-12a8121b4da21ebf.png)

 说明我们的HTTP服务器已经配置成功了！

搭建HTTPS服务器
----
我们只需要在HTTP服务器的基础上配置HTTPS就可以了。

* **创建自签名SSL/TLS证书**  
首先使用OpenSSL创建私钥文件

 ```
 openssl genrsa -out mywebsite.key 2048
 ```
然后利用私钥创建自签名证书

 ```
 openssl req -new -x509 -key mywebsite.key -out mywebsite.cer
 ```
需要自己填写一些证书信息，如下图：

![](http://upload-images.jianshu.io/upload_images/3145770-2fe7e964244b672f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 红框标注选项**最好填写虚拟主机域名，如果不按要求填写也可以使用**，同理我们创建另一个虚拟主机**mywebsite2.com**证书。

* **修改服务器配置**  
去掉`#Include /private/etc/apache2/extra/httpd-ssl.conf`前面的`#`,
去掉`#LoadModule ssl_module libexec/apache2/mod_ssl.so`前面的`#`,
去掉`#LoadModule socache_shmcb_module libexec/apache2/mod_socache_shmcb.so`前面的`#`。

* **修改httpd-ssl.conf配置**  
打开`/private/etc/apache2/extra/httpd-ssl.conf`文件，找到对应的`SSLCertificateFile`和`SSLCertificateKeyFile`字段，加入如下代码：

 ```
 SSLCertificateFile "/Users/libo/apache_server/mywebsite/mywebsite.cer"
 SSLCertificateFile "/Users/libo/apache_server/mywebsite2/mywebsite2.cer"
 ```

 ```
 SSLCertificateKeyFile "/Users/libo/apache_server/mywebsite/mywebsite.key"
 SSLCertificateKeyFile "/Users/libo/apache_server/mywebsite2/mywebsite2.key"
 ```

* **修改虚拟主机配置**   
更改虚拟主机默认端口号为`443`，配置服务器证书`SSLCertificateFile`和私钥`SSLCertificateKeyFile`，如下：

 ```
 <VirtualHost *:443>
     # ServerAdmin webmaster@dummy-host.example.com
     DocumentRoot "/Users/libo/apache_server/mywebsite"
     ServerName mywebsite.com
     ErrorLog "/private/var/log/apache2/mywebsite.com-error_log"
     CustomLog "/private/var/log/apache2/mywebsite.com-access_log" common

     SSLCertificateFile    "/Users/libo/apache_server/mywebsite/mywebsite.cer"
     SSLCertificateKeyFile "/Users/libo/apache_server/mywebsite/mywebsite.key"

     <Directory />
                 Options Indexes FollowSymLinks MultiViews
                 AllowOverride all
                 Order deny,allow
                 Allow from all
     </Directory>

 </VirtualHost>

 <VirtualHost *:443>
     # ServerAdmin webmaster@dummy-host2.example.com
     DocumentRoot "/Users/libo/apache_server/mywebsite2"
     ServerName mywebsite2.com
     ErrorLog "/private/var/log/apache2/mywebsite2.com-error_log"
     CustomLog "/private/var/log/apache2/mywebsite2.com-access_log" common

     SSLCertificateFile    "/Users/libo/apache_server/mywebsite2/mywebsite2.cer"
     SSLCertificateKeyFile "/Users/libo/apache_server/mywebsite2/mywebsite2.key"

     <Directory />
                 Options Indexes FollowSymLinks MultiViews
                 AllowOverride all
                 Order deny,allow
                 Allow from all
     </Directory>

 </VirtualHost>
 ```

* **验证**  
重启服务器`sudo apachectl restart`，浏览器访问，如下图：

![](http://upload-images.jianshu.io/upload_images/3145770-3a6a826a8fe1da73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/3145770-58ed17ea9e97439e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 可以看到我们的**HTTPS**服务器已经搭建成功了，尽管不是“*安全*”的访问！接下来我们要进行一些操作使访问变得“*安全*”。

* **创建安全的HTTPS访问**  
所谓创建安全的HTTPS访问，就是让浏览器信任服务器颁发的公钥证书，因为我们的证书没有添加到浏览器信任列表，所以浏览器会提示我们“*不安全*”。废话不多说，首先创建CA根证书：
  
 **创建CA私钥：**  

 ```
 openssl genrsa -des3 -out Apache_CA.key 2048
 ```

 这里使用-des3进行加密，需要4~1023位密码。  
 
 **创建CA证书：** 
   
 ```
 openssl req -new -x509 -days 365 -key Apache_CA.key  -out Apache_CA.cer
 ```

 这里需要输入刚才设置的CA私钥密码并填写一些证书信息。
 
![](http://upload-images.jianshu.io/upload_images/3145770-edea43e3662a8d20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **创建服务器证书私钥：**  
 
 ```
 openssl genrsa -out mywebsite.key 2048
 ```
 
 **生成证书请求文件CSR：**  

 ```
 openssl req -new -key mywebsite.key -out mywebsite.csr
 ```

 这里同样需要填写一些证书信息，同样，红框标注选项**最好填写虚拟主机域名**。
 
![](http://upload-images.jianshu.io/upload_images/3145770-be6b1e08aaf8c5bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **生成证书配置文件v3.ext：**   

 ```
 authorityKeyIdentifier=keyid,issuer
 basicConstraints=CA:FALSE
 keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
 subjectAltName = @alt_names

 [alt_names]
 DNS.1 = mywebsite.com
 ```
 
 `DNS.1`要写配置证书的服务器域名。

 **用自己的CA签发证书：**  

 ```
 openssl x509 -req -in mywebsite.csr -CA Apache_CA.cer -CAkey Apache_CA.key -CAcreateserial -out mywebsite.cer  -days 365 -sha256 -extfile v3.ext
 ```
 
 然后需要输入生成Apache_CA.key时设置的密码。
 
![](http://upload-images.jianshu.io/upload_images/3145770-539aa46048560fe9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 恭喜你，证书已经创建成功！然后我们按照前面提到的步骤替换掉服务器的证书，重启服务器`sudo apachectl restart`。
 
 **设置系统始终信任CA证书：**  
 双击点开`Apache_CA.cer` ，然后在钥匙串中找到该证书，`右键->显示简介->信任`，设置为始终信任。然后分别再次访问网址`https://mywebsite.com/index`和`https://mywebsite1.com/index`。访问结果如下：
 
![](http://upload-images.jianshu.io/upload_images/3145770-79f0b8a2af28264f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/3145770-ef92558717820e39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 恭喜你！`mywebsite.com`服务器的访问已经变成“*安全*”的了，同理可以配置`mywebsite2.com`服务器。但是，你要注意一点，这种“*安全*”只是相对的安全，或者说是一种假象，因为你的CA证书是自己创建颁发的，别人也可以通过窃取你的CA私钥或者其他方式来伪造证书。更为稳妥的办法还是要向正规的证书颁发机构去申请私钥和证书，这点一定要注意。

搭建正向代理服务器
----
在搭建正向代理服务器之前，我们先说下正向代理和反向代理的区别。我们平时访问网上的资源，绝大多数情况是要经过各种（正向或反向或其他）代理的，只是这对用户来说是*无感知*的。正向代理可以理解为一层*跳板*，我们访问资源的目的IP地址是**服务器**，只是经过正向代理这个节点。而反向代理是我们访问资源的目的IP地址就是**反向代理服务器**，反向代理服务器和最终的服务器进行交互，获取资源。下图可以很清晰的展示这两种关系：

![](http://upload-images.jianshu.io/upload_images/3145770-4894f2597f8dfb0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/3145770-08271725d20229fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  
下面我们开始配置正向代理服务器。
 
* **修改服务器配置**  
打开配置文件`/private/etc/apache2/httpd.conf`，去掉以下`module`前面的`#`。

 ```
 #LoadModule proxy_module libexec/apache2/mod_proxy.so
 #LoadModule proxy_connect_module libexec/apache2/mod_proxy_connect.so
 #LoadModule proxy_ftp_module libexec/apache2/mod_proxy_ftp.so
 #LoadModule proxy_http_module libexec/apache2/mod_proxy_http.so
 ```
 
 通过以上`module`的命名我们可以了解到`mod_proxy.so`是基础的代理配置，`mod_proxy_http.so`支持`HTTP`请求，`mod_proxy_ftp.so`支持`FTP`请求，`mod_proxy_connect.so `支持`HTTPS`请求（`HTTPS`请求头和报文是加密的，代理服务器不能通过识别请求头来获取目的服务器的地址，所以在最开始建立连接时代理服务器需要打开一条从客户端到服务器的端到端`connect`通道）。

* **修改虚拟主机配置**   
将虚拟主机端口号改为`80`，加入正向代理设置。  

 ```
 <VirtualHost *:80>
     # ServerAdmin webmaster@dummy-host.example.com
     DocumentRoot "/Users/libo/apache_server/mywebsite"
     ServerName mywebsite.com
     ErrorLog "/private/var/log/apache2/mywebsite.com-error_log"
     CustomLog "/private/var/log/apache2/mywebsite.com-access_log" common

     SSLCertificateFile    "/Users/libo/apache_server/mywebsite/mywebsite.cer"
     SSLCertificateKeyFile "/Users/libo/apache_server/mywebsite/mywebsite.key"

     <Directory />
                 Options Indexes FollowSymLinks MultiViews
                 AllowOverride all
                 Order deny,allow
                 Allow from all
     </Directory>

     #正向代理设置
     ProxyRequests On
     ProxyVia Full

     <Proxy *>
         Order deny,allow
         #Deny from all
         Allow from all
     </Proxy>

 </VirtualHost>
 ```
 
 `ProxyVia Full`可以为我们打出最详细的代理服务器信息。

* **创建自动代理配置**   
为了方便测试，我们创建`mywebsite.pac`文件来配置代理。

 ```
 function FindProxyForURL(url, host) {
     if (host == "www.jianshu.com") {
         return "PROXY 127.0.0.1:80";
     }
     return 'DIRECT;';
 }
 ```
 我们有选择的只要求访问简书`www.jianshu.com`域名时经过我们的代理服务器，其他域名进行`DIRECT`直连。
 
* **配置代理配置**
打开系统偏好设置**->**网络**->**高级**->**代理，选中自动代理配置，配置`mywebsite.pac`文件的路径地址，然后点击好**->**应用。

* **验证**   
浏览器打开简书首页`http://www.jianshu.com/`，打开开发者模式：

![](http://upload-images.jianshu.io/upload_images/3145770-4802d987dce0d4b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)
 
 通过`Via`字段，我们发现这次请求经过了我们的代理服务器，说明我们的配置成功了！

搭建反向代理服务器
----
* **修改服务器配置**  
我们需要用`PHP`脚本来测试反向代理，`Apache`服务器自身支持`PHP`，只需要打开配置文件`/private/etc/apache2/httpd.conf`，去掉`#LoadModule php5_module libexec/apache2/libphp5.so`前面的`#`。

* **修改虚拟主机配置**   
我们把`mywebsite.com`和`mywebsite2.com`的默认端口号改为`80`，让`mywebsite2.com`作为反向代理服务器，`mywebsite.com`作为原始服务器，通过访问反向代理服务器间接访问原始服务器资源。配置代码如下：

 ```
 <VirtualHost *:80>
     # ServerAdmin webmaster@dummy-host2.example.com
     DocumentRoot "/Users/libo/apache_server/mywebsite2"
     ServerName mywebsite2.com
     ErrorLog "/private/var/log/apache2/mywebsite2.com-error_log"
     CustomLog "/private/var/log/apache2/mywebsite2.com-access_log" common

     SSLCertificateFile    "/Users/libo/apache_server/mywebsite2/mywebsite2.cer"
     SSLCertificateKeyFile "/Users/libo/apache_server/mywebsite2/mywebsite2.key"

     <Directory />
                 Options Indexes FollowSymLinks MultiViews
                 AllowOverride all
                 Order deny,allow
                 Allow from all
     </Directory>

     #反向代理设置
     ProxyPass / http://mywebsite.com/
     ProxyPassReverse / http://mywebsite.com/

 </VirtualHost>
 ```

 `ProxyPass / http://mywebsite.com/`是把所有访问当前主机`http://mywebsite2.com/`的请求转发给`http://mywebsite.com/`主机，至于`ProxyPassReverse / http://mywebsite.com/`的作用，我们稍后再说。

* **验证**  
我们在`"/Users/libo/apache_server/mywebsite"`路径下创建两个`PHP`文件:

 **redirect.php**

 ```
 <?php

   function redirect($url)
   {
    header("Location: $url");
    exit();
   }

   $url = "http://mywebsite.com/test.php";
   redirect($url);
  
 ?>
 ```
 
 **test.php**

 ```
 <?php

 phpinfo();
  
 ?>
 ```
 
 重启服务器，访问`http://mywebsite2.com/redirect.php`。
 
![](http://upload-images.jianshu.io/upload_images/3145770-180172fd75ffbbfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)
 
 可以看到请求的`Request URL`的`host`还是`mywebsite2.com`，但是它确实是由 `mywebsite.com`来处理的，通过访问`mywebsite.com`资源`redirect.php`文件，进而重定向到`test.php`文件。说明我们反向代理服务器已经搭建成功了！  
 
 现在我们介绍下`ProxyPassReverse `的作用，我们把配置文件的这一项配置去掉，重启服务器再次访问`http://mywebsite2.com/redirect.php`。
 
![](http://upload-images.jianshu.io/upload_images/3145770-3a632939a0791807.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)
 
 和上图对比可以看到请求的`Request URL`的`host`是`mywebsite.com`而不是 `mywebsite2.com`，这是因为配置了`ProxyPassReverse`后，`mywebsite.com/redirect.php`在重定向到`mywebsite.com/test.php`时，Apache会将它调整回 `mywebsite2.com/test.php`， 然后Apache再将`mywebsite2.com/test.php` 转发给`mywebsite.com/test.php`。所以说配置了`ProxyPassReverse`后，即使在请求过程中发生了重定向，Apache也会帮你擦去这些*痕迹*。

总结
----
以上都是我对`Apache`服务器的基础配置，简单的实现了预期功能。服务器配置很细碎繁琐，能根据不同代码实现更为复杂精细的配置，更为详细的功能具体请参考[官方文档](http://httpd.apache.org/docs/2.4/)，本人没有做深入研究。本文如有错误，希望指正，一起学习，共同进步，不胜感激！

参考链接
----
<http://www.liuchungui.com/blog/2015/09/25/zi-jian-zheng-shu-pei-zhi-httpsfu-wu-qi/>  
<http://beyondvincent.com/2014/03/17/2014-03-17-five-tips-for-using-self-signed-ssl-certificates-with-ios/> 
<http://www.cnblogs.com/zemliu/archive/2012/04/18/2454655.html> 
<http://httpd.apache.org/docs/2.4/>



> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
