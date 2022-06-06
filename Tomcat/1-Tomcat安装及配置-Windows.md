# Tomcat 安装及配置（Windows）

[TOC]

## 1. Tomcat 版本

登录 [Apache Tomcat ](http://tomcat.apache.org/) 官网，点击[版本选择链接](https://tomcat.apache.org/whichversion.html)，可以看到Tomcat 提供多个版本供用户下载。个人习惯使用Tomcat8 或 Tomcat9 版本，JDK8 环境。

 ![Tomcat版本选择](http://image.xgoding.top/tomcat_whichversion.jpg)

## 2. Tomcat下载

点击对应版本，显示下载页面，如下图所示：

![Tomcat Download](http://image.xgoding.top/tomcat_download.jpg)

Tomcat版本下载分为：

* 二进版本
  * 核心版本
  * 全文档
  * Web应用发布工具
  * 嵌入式版本
* 源码版本

这里，我们选择64位压缩包版本。如需下载其他版本，点击 **Archives** 选择对应版本下载即可。

 ![tomcat_archives_1](http://image.xgoding.top/tomcat_archives_1.jpg)

![tomcat-archives-2](http://image.xgoding.top/tomcat_archives_2.jpg)

## 3.Tomcat 安装

下载完成后，解压到指定文件夹即可。

 ![tomcat_install](http://image.xgoding.top/tomcat_install.jpg)

## 4. JDK环境配置

由于Tomcat运行需要JDK环境支持，因此需要配置JDK环境，本文不赘述JDK安装及配置过程。打开命令工具输入 `java -version` 检查JDK安装情况。

 ![java_version](http://image.xgoding.top/java_version.jpg)

## 5.Tomcat环境配置

打开电脑环境变量，新建`CATALINA_HOME`变量，如下图：

 ![catalina_home](http://image.xgoding.top/catalina_home.jpg)

编辑**path**环境变量，新增`%CATALINA_HOME%\bin`路径：

 ![path_catalina_bin](http://image.xgoding.top/path_catalina_bin.jpg)

## 6.Tomcat启动

1. 双击运行`bin\startup.bat`启动Tomcat

2. cmd 中运行`%CATALINA_HOME%\bin\startup.bat`启动Tomcat

Tomcat启动完成后如图所示：

 ![tomcat_startup](http://image.xgoding.top/tomcat_startup.jpg)

访问本地8080端口，http://localhost:8080/：

![tomcat_web_8080](http://image.xgoding.top/tomcat_web_8080.jpg)

Tomcat启动成功。

## 7.Tomcat日志乱码

通过窗口方式启动Tomcat后我们发现中文乱码，由于window命令行窗口输出中文默认与操作系统保持一致，即采用**GBK**编码方式，而Tomcat默认使用**UTF-8**编码方式，导致输出中文乱码。解决这个问题只需要修改配置**\conf\logging.properties** 中编码方式即可。

 ![tomcat_conf_console_encoding](http://image.xgoding.top/tomcat_conf_console_encoding.jpg)

重新运行Tomcat，如图所示：

 ![tomcat_starup](http://image.xgoding.top/tomcat_startup_2.jpg)

## 8.Tomcat 注册Windows服务

### 8.1 服务注册

打开`%CATALINA_HOME%\bin`路径，运行`service install <服务名>` 安装服务：

 ![tomcat_service_install_cmd](http://image.xgoding.top/tomcat_service_install_cmd.png)

打开Windows服务管理器，查看安装完成的服务：

 ![tomcat_windows_service](http://image.xgoding.top/tomcat_service.jpg)

### 8.2 服务卸载

同样，打开命令行，输入`service remove`进行卸载。

 ![tomcat_service_remove](http://image.xgoding.top/tomcat_service_remove.jpg)

打开服务管理器，发现刚刚注册的服务已经被删除了。

