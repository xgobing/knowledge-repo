# Gradle 安装配置及镜像配置

[TOC]



## Gradle 安装

1. 点击下载

   [管网地址](https://gradle.org/releases/)

2. 配置环境变量

   解压Gradle压缩包到指定目录，如 `D:\DevEnv\gradle\gradle-7.3-bin\gradle-7.3`，并创建环境变量 `GRADLE_HOME`.

   ![GRADLE_HOME](http://image.xgoding.top/gradle%2FGRADLE_HOME.png)  

   添加到 `Path` 环境变量：

   ![GRADLE_HOME_PATH](http://image.xgoding.top/gradle%2FGRADLE_HOME_PATH.png)  

   配置 `GRADLE_USER_HOME` 环境变量，默认值为 :   `%USER_HOME%`/.gradle  ，这个配置的作用和在IDEA中配置的gradle user home相同的，可选配。

   此处变量值修改为：`D:\DevEnv\gradle\repo`

   ![GRADLE_USER_HOME](http://image.xgoding.top/gradle%2FGRADLE_USER_HOME.png)   

   

   3. 测试 

   命令行输入 `gradle -v` 查看版本信息：

   ![GRADLE_CMD_V](http://image.xgoding.top/gradle%2FGRADLE_CMD_V.png)  

   命令行输入 `gradle --status` 查看运行状态：

   ![GRADLE_CMD_STATUS](http://image.xgoding.top/gradle%2FGRADLE_CMD_STATUS.png)  

   

   

   ## Gradle全局镜像配置  

   为了提高Gradle编译及下载效率，这里通过配置 [阿里云](https://developer.aliyun.com/mvn/guide) 镜像源：

   1. `GRADLE_USER_HOME` 目录下创建 `init.gradle` 文件：
   
   ```
   allprojects{
       repositories {
           def ALIYUN_CENTRAL_URL = 'https://maven.aliyun.com/repository/central'
   	def ALIYUN_JCENTER_URL = 'https://maven.aliyun.com/repository/public'
           def ALIYUN_PUBLIC_URL = 'https://maven.aliyun.com/repository/public'
           def ALIYUN_GOOGLE_URL = 'https://maven.aliyun.com/repository/google'
           def ALIYUN_GRADLE_PLUGIN_URL = 'https://maven.aliyun.com/repository/gradle-plugin'
   	def ALIYUN_SPRING_URL = 'https://maven.aliyun.com/repository/spring'
   	def ALIYUN_SPRING_PLUGIN_URL = 'https://maven.aliyun.com/repository/spring-plugin'
   	def ALIYUN_GRAILS_CORE_URL = 'https://maven.aliyun.com/repository/grails-core'
   	def ALIYUN_APACHE_SNAPSHOTS_URL = 'https://maven.aliyun.com/repository/apache-snapshots'
           all { ArtifactRepository repo ->
               if(repo instanceof MavenArtifactRepository){
                   def url = repo.url.toString()
                   if (url.startsWith('https://repo1.maven.org/maven2/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_CENTRAL_URL."
                       remove repo
                   }
                   if (url.startsWith('https://jcenter.bintray.com/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_PUBLIC_URL."
                       remove repo
                   }
   		if (url.startsWith('https://maven.google.com/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_GOOGLE_URL."
                       remove repo
                   }
                   if (url.startsWith('https://plugins.gradle.org/m2/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_GRADLE_PLUGIN_URL."
                       remove repo
                   }
   		if (url.startsWith('http://repo.spring.io/libs-milestone/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_SPRING_URL."
                       remove repo
                   }
   		if (url.startsWith('http://repo.spring.io/plugins-release/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_SPRING_PLUGIN_URL."
                       remove repo
                   }
   		if (url.startsWith('https://repo.grails.org/grails/core')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_GRAILS_CORE_URL."
                       remove repo
                   }
   		if (url.startsWith('https://repository.apache.org/snapshots/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_APACHE_SNAPSHOTS_URL."
                       remove repo
                   }
   		if (url.startsWith('https://dl.google.com/dl/android/maven2/')) {
                       project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_GOOGLE_URL."
                       remove repo
                   }
               }
           }
           maven { url ALIYUN_CENTRAL_URL }
           maven { url ALIYUN_JCENTER_URL }
           maven { url ALIYUN_PUBLIC_URL }
   	maven { url ALIYUN_GOOGLE_URL }
           maven { url ALIYUN_GRADLE_PLUGIN_URL }
   	maven { url ALIYUN_SPRING_URL }
   	maven { url ALIYUN_SPRING_PLUGIN_URL }
   	maven { url ALIYUN_GRAILS_CORE_URL }
   	maven { url ALIYUN_APACHE_SNAPSHOTS_URL }
       }
   }
   ```

   2. 创建配置文件 `gradle.properites`

      新增 `org.gradle.daemon=true` 启动守护进程，保存后自动生效。  

   

   

   ## Gradle单个项目镜像配置
   
   只需修改项目配置文件 `build.gradle` ，增加相关仓库镜像：
   
   ```
   buildscript {
       repositories {
           maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
           maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
       }
       dependencies {
           classpath 'com.android.tools.build:gradle:2.2.3'
   
           // NOTE: Do not place your application dependencies here; they belong
           // in the individual module build.gradle files
       }        
   }
   
   allprojects {
       repositories {
           maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
           maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
       }
   }
   
   ```
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   