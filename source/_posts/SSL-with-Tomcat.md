---
title: SSL with Tomcat
date: 2017-11-17 18:20:05
categories: DevOps
---
## Prepare the Certificate Keystore
### Certificate Keystore Type
> Tomcat当前支持 JKS，PKCS11，PKCS12（v5.0/5.0+）书格式。JKS为JAVA标准的『Java KeyStore』格式，是由JDK的keytool命令行工具生成。PKCS12是互联网标准格式，由OpenSSL与Microsoft's Key-Manager共同维护。

创建一个自签名的JKS keystore
- Windows:

  ``` bash
  "%JAVA_HOME%\bin\keytool" -genkey -alias tomcat -keyalg RSA -keystore \path\to\my\keystore
  ```

- Unix:

  ``` bash
  $JAVA_HOME/bin/keytool -genkey -alias tomcat -keyalg RSA -keystore /path/to/my/keystore
  
  ```

> 目前大多数云服务商都提供系统自动生成keystore。

<!-- more -->

### Certificate Keystore Transform
使用java jdk将PFX格式证书转换为JKS格式证书(windows环境注意在%JAVA_HOME%/jdk/bin目录下执行)

``` bash
$ keytool -importkeystore -srckeystore your-name.pfx -destkeystore your-name.jks -srcstoretype PKCS12 -deststoretype JKS
```

回车后输入JKS证书密码和PFX证书密码，强烈推荐将JKS密码与PFX证书密码相同，否则可能会导致Tomcat启动失败。

## Edit the Tomcat Configuration File

Tomcat使用2种不同的SSL实现方式：
  * 与Java runtime(since 1.4)一起提供的`JSSE`实现
  * 默认使用OpenSSL引擎的`APR`实现


若在server.xml文件中使用通用的`protocol="HTTP/1.1"`来配置Connector，则由Tomcat自动选择SSL实现方式。在安装了 Tomcat native libraryr 情况下，将使用APR SSL来实现，否则使用Java JSSE来实现。由于APR与JSSE实现有很大的差异，建议避免自动选择来实现。

### JSSE connector
定义JSSE connector，无论是否加载APR库，使用以下方式之一：

``` xml
<!-- Define a HTTP/1.1 Connector on port 8443, JSSE NIO implementation -->
<Connector protocol="org.apache.coyote.http11.Http11NioProtocol"
           port="8443" .../>

OR:

<!-- Define a HTTP/1.1 Connector on port 8443, JSSE BIO implementation -->
<Connector protocol="org.apache.coyote.http11.Http11Protocol"
           port="8443" .../>

```
> **The BIO and NIO connectors use JSSE whereas the APR/native connector uses APR.**

### APR connector
定义APR connector(APR库必须可用)

``` xml
<!-- Define a HTTP/1.1 Connector on port 8443, APR implementation -->
<Connector protocol="org.apache.coyote.http11.Http11AprProtocol"
           port="8443" .../>
```
           
在使用APR时，可以选择配置OpenSSL作为引擎：

``` xml
<Listener className="org.apache.catalina.core.AprLifecycleListener"
          SSLEngine="someengine" SSLRandomSeed="somedevice" />
```

ARP默认值：

``` xml
<Listener className="org.apache.catalina.core.AprLifecycleListener"
          SSLEngine="on" SSLRandomSeed="builtin" />
```
要在APR下使用SSL，要确保SSLEngine属性设置为非关闭状态，默认值是打开的。如果你指定另一个值，它必须是一个有效的引擎名称。

~~为什么网上有文章说要注释掉~~

``` xml
<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
```

~~原因在`自动选择`~~

## Configure SSL

+ With JSSE

``` xml
<!-- Define a SSL Coyote HTTP/1.1 Connector on port 8443 -->
<Connector
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           port="8443" maxThreads="200"
           scheme="https" secure="true" SSLEnabled="true"
           keystoreFile="${user.home}/.keystore" keystorePass="changeit"
           clientAuth="false" sslProtocol="TLS"/>
```

+ With APR

``` xml
<!-- Define a SSL Coyote HTTP/1.1 Connector on port 8443 -->
<Connector
           protocol="org.apache.coyote.http11.Http11AprProtocol"
           port="8443" maxThreads="200"
           scheme="https" secure="true" SSLEnabled="true"
           SSLCertificateFile="/usr/local/ssl/server.crt"
           SSLCertificateKeyFile="/usr/local/ssl/server.pem"
           SSLVerifyClient="optional" SSLProtocol="TLSv1+TLSv1.1+TLSv1.2"/>
```

**Reference**
[HTTP connector](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html#SSL%20Support)
[Apache HOW-TO](https://tomcat.apache.org/tomcat-7.0-doc/ssl-howto.html)
