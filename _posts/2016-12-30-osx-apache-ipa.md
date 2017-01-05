---
layout: post
title:  "OS X + Apache 本地发布安装ipa"
date:   2016-12-30 10:10:00 +0800
categories: jekyll update
---
软件环境：OS X EI Capitan v10.11.6   
　　　　　Apache/2.4.18 (Unix)  

OS X自带Apache，查看是否开启可以打开活动监视器搜索httpd，如果未打开可以使用命令 sudo apachectl start 开启

前置条件：  
ipa需要企业开发者证书打包  
Apache需要支持https

### 这里先说一下Apache如何开启https，主要有两部分：  

#### 一、生成ssl自签名证书

1. 生成主机密钥  
**mkdir /etc/apache2/ssl  
cd    /etc/apache2/ssl  
sudo ssh-keygen -f server.key**  
使用sudo是目录需要管理员权限，需要输入管理员密码  
生成秘钥也会提示设置密码，我这里置空

2. 生成证书请求文件  
**sudo openssl req -new -key server.key -out request.csr**  
执行结果  
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.  
\-----  
Country Name (2 letter code) [AU]:.    
State or Province Name (full name) [Some-State]:.   
Locality Name (eg, city) []:.  
Organization Name (eg, company) [Internet Widgits Pty Ltd]:.  
Organizational Unit Name (eg, section) []:  
**Common Name (e.g. server FQDN or YOUR name) []:gzl.com**                         
Email Address []:.  
Please enter the following 'extra' attributes  
to be sent with your certificate request  
A challenge password []:  
An optional company name []:.  
这里主要是填写自签名证书的一些基本信息，其中 **Common Name** 这里设置的值需要是你本地服务器将要使用的域名，涉及到证书和域名是否匹配的问题，这设置的是 **gzl.com**
3. 生成ssl证书  
用上一步生成的文件生成ssl证书  
**sudo openssl x509 -req -days 365 -in request.csr -signkey server.key -out server.crt**

*通过以上步骤，/etc/apache2/ssl目录下应该有4个文件，分别是：  
request.csr、	server.crt、	server.key、	server.key.pub，其中 **sever.crt** 在下面使用iphone访问下载地址是否信任的时候会用到*

#### 二、配置Apache支持https  
  
1. /etc/apache2/httpd.conf，编辑这个文件去掉下面三行前面的 '#'  
**LoadModule ssl_module libexec/apache2/mod_ssl.so  
Include /private/etc/apache2/extra/httpd-ssl.conf  
Include /private/etc/apache2/extra/httpd-vhosts.conf**
 
2. /etc/apache2/extra/httpd-ssl.conf，编辑这个文件去掉下面两行前面的 '#'  
**SSLCertificateFile "/private/etc/apache2/ssl/server.crt"  
SSLCertificateKeyFile "/private/etc/apache2/ssl/server.key"**
3. /etc/apache2/extra/httpd-vhosts.conf，编辑这个文件，在文件末尾添加  
\<VirtualHost *:443>  
　SSLEngine on  
　SSLCipherSuite ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP:+eNULL  
　SSLCertificateFile /etc/apache2/ssl/server.crt  
　SSLCertificateKeyFile /etc/apache2/ssl/server.key  
　**ServerName gzl.com**  
　**DocumentRoot "/Library/WebServer/Documents/"**  
\</VirtualHost>  
其中  
**ServerName**需要与上面生成自签名证书请求csr文件填写的 **Common Name** 相一致  
**DocumentRoot**填写的Apache的默认目录**/Library/WebServer/Documents/**

到这里就配置完了，检查配置  
**apachectl configtest**
  
遇到的问题：    
  
1. AH00526: Syntax error on line 92 of /private/etc/apache2/extra/httpd-ssl.conf:
SSLSessionCache: 'shmcb' session cache not supported (known names: ). Maybe you need to load the appropriate socache module (mod_socache_shmcb?).
2. AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using zl-mac2.local. Set the 'ServerName' directive globally to suppress this message

解决方案：
  
1. /etc/apache2/httpd.conf，编辑这个文件去掉下面这一行前面的 '#'  
**LoadModule socache_shmcb_module libexec/apache2/mod_socache_shmcb.so** 
2. 是因为httpd.conf文件中的ServerName没有配置，处于缺省状态。  
只需要在apache安装目录/etc/apache2/httpd.conf文件中启用ServerName配置指令即可。  
apache的配置文件httpd.conf中默认是存在类似的指令的，不过在该指令前添加了＃号，注释掉了该句，我们只需要模仿着增加一行：  
**ServerName gzl.com:80 ** 

最后出现 ** Syntax OK** 即说明配置正确，重启Apache就可以了  
**sudo apachectl restart**

想要使用域名访问本地的服务器需要编辑 /etc/hosts 文件，绑定一下域名，这里添加的是：  
**127.0.0.1　　gzl.com**  
这时候在safari中访问https://gzl.com，会提示证书不信任，双击生成的证书文件server.crt安装，选择始终信任，刷新后在地址栏即可以看到安全的标志，如果没修改过本地服务器首页内容的话，会默认显示It works！

### 下面说一下ipa相关的静态html页面和plist设置 

#### index.html:  

	<html><meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<body>
	<a href = "itms-services://?action=download-manifest&url=https://gzl.com/download/speex.plist">install ipa test</a>
	</body>
	</html>

#### speex.plist

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
	<plist version="1.0">
	<dict>
		<key>items</key>
		<array>
			<dict>
				<key>assets</key>
				<array>
					<dict>
						<key>kind</key>
						<string>software-package</string>
						<key>url</key>
						<string>https://gzl.com/download/speex.ipa</string>
					</dict>
				</array>
				<key>metadata</key>
				<dict>
					<key>bundle-identifier</key>
					<string>com.zlw.iosweibo</string>
					<key>bundle-version</key>
					<string>1.0</string>
					<key>kind</key>
					<string>software</string>
					<key>title</key>
					<string>Speex</string>
				</dict>
			</dict>
		</array>
	</dict>
	</plist>
	
***注意：index.html 和 speex.plist 中用到的链接 https://gzl.com/download/speex.ipa ，是我本地服务器存储ipa的位置（ipa和plist文件需要在同一个目录下），DocumentRoot "/Library/WebServer/Documents/"，此目录下主要的文件目录如下：***

	/Documents
		/download
			speex.plist
			speex.ipa
		index.html	
		
### 最后说一下iPhone如何访问本地服务器安装ipa
访问本地服务器使用了mac充当代理的方法，步骤如下：

1. mac和iPhone连接同一个wifi，这里使用现有的charels开启代理，打开iPhone 设置--无线局域网，设置wifi的HTTP代理，选手动输入mac连接wifi的ip地址，端口号8888（charels默认），不明白的可以参考[MAC上charles使用教程总结](http://www.jianshu.com/p/18449f5f9d1c)，*当然如果有其他可以在mac上开启代理的软件也是可以的*
2. 把上面生成的ssl证书 **server.crt** 以邮件的形式发送到手机上，点击安装
3. 使用safari访问下载地址 *https://gzl.com* 点击页面上的链接等待安装完成即可

注意事项：

1. 必须使用safari访问
2. iOS 9即以上打开企业开发者app是不能直接信任了，需要在 **设置--通用--描述文件与设备管理** 选择相应的证书点击信任  
