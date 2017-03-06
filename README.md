# Maven私有服务器搭建与Android Studio Library管理（Maven Server Creation And Android Library Management）

此工程包含了与这篇文章相应的demo实例。方便你马上上手。

（初版文档，文档内容需要随时更改和补充）

##引言
公司需要有自己的library仓库，避免重复造车轮子，方便需要时随时使用。一个方法就是建立自己的私有repository。本文档教你如下内容：
* 如何搭建Nexus Maven私服
* 在Android Studio中上传私有library
* 使用私有library

##Nexus下载和安装（如果已经安装好服务器，省略此步骤）
本地仓库需要有自己的仓库管理工具。Sonatype Nexus和Artifactory都是不错的选择。本文选择Sonatype Nexus。下载地址为：
https://www.sonatype.com/download-oss-sonatype

请根据你的操作系统进行选择。下载解压后，你可以看到如下的文件夹：

在Console中（windows下为cmd），键入：
```
Windows：
将nexus.exe设置成以管理员身份运行
cd {nexus解压后目录/nexus-版本号/bin/}
nexus.exe /install
nexus.exe /start
Mac：
cd {nexus解压后目录/nexus-版本号/bin/}
./nexus run

```
如果有疑问请参考：
http://books.sonatype.com/nexus-book/reference3/install.html#service-windows

##打开私服管理器
在浏览器中，输入http://localhost:8081，如果一切顺利，你可以看到Nexus Repository Manager管理页面。

Tips：你也可以使用花生壳等工具，将其部署到外网中去（通过内网映射），这样你就可以用外网地址访问私服服务器了。

##创建私有仓库
私服管理页面中，你首先会进入guest账户页面，这个账户你还不能新建repository。这里点击右上角的sign in

输入管理员密码

默认为admin,密码admin123。登陆后，点击设置按钮 ，点击Repository-Repository-Create Repository

选择maven2（hosted）类型。当然，你也可以选择别的类型的repository。这里选择maven2是为了和android studio的maven插件配合使用，也是推荐的选择。

输入repository的名字。Deployment policy选择Allow redeploy，否则library的提交就只能进行一次，不能重复提交。点击Create repository。

至此私有Maven库就创建完毕了。

##创建私有仓库
Library的创建是为了解决一个实际问题，可以完全是自己新建的项目，可以来自局域网的svn，也可以是在github上委托的项目。

Tips:
如何github上的项目创建，请看
https://www.londonappdeveloper.com/how-to-use-git-hub-with-android-studio/。

* 首先，我们在Android Studio中新建一个project，点击File-New-New Project。
* 新建一个Module，选择Android Library，指定Library Name，点击Finish。
* 根据项目需要，编写Library Module。对于公司编写的library而言，这里应该执行版本管理，利用svn或者git等等。

Tips：library完成了某一milestone，或者进行了一遍迭代时，才应该上传代码到maven库。请注意区分这两种不同的提交。
* 为了gradle文件的模块性，新建nexus_maven.gradle文件，输入如下代码：
```
apply plugin: 'maven'

task androidJavadocs(type: Javadoc) {    
  source = android.sourceSets.main.java.srcDirs    
  classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
}

task androidJavadocsJar(type: Jar, dependsOn: androidJavadocs) {
  classifier = 'javadoc'    
  from androidJavadocs.destinationDir
}

task androidSourcesJar(type: Jar) {    
  classifier = 'sources'    
  from android.sourceSets.main.java.srcDirs
}

artifacts {    
  archives androidSourcesJar    
  archives androidJavadocsJar
}

uploadArchives {    
  repositories {        
    mavenDeployer {            
      repository(url: "YourMavenRepositoryUrl") {                            
        authentication(userName: "admin", password: "admin123")            
      }            
      pom.project {                
        name 'YourProjectName'                
        version '1.0.0'                
        artifactId 'yourartifactid'                
        groupId 'com.company'                
        packaging 'aar'                
        description 'Your Project Description'            
      }       
    }    
  }
}
```
这段代码首先引用maven插件；然后定义了2个任务：androidSourcesJar和androidJavadocsJar，这两个任务分别用于对Java sources打包和Java doc进行打包；接着我们对uploadArchives.repositories闭包进行一些配置，包括仓库的url地址，比如http://localhost:8081/repository/android-lib，上传所需的用户名和密码，以及pom属性。

* 在module的build.gradle文件最后添加
```
apply from: './nexus_maven.gradle'
```
这样，代码就可以准备上传到Maven私服了。

## 上传本地代码到Maven库
点击Android Studio右侧的Gradle projects。双击uploadArchives上传代码。在Run Console中，查看是否成功。在Maven私服后台中，点击Browse server contents-Browse-Components，你应该可以看到刚才上传的repository。

Tips：一般而言，maven库的提交者维护着整个library的版本，不应该由library的开发者执行，repository url用localhost应该已经足够了。

##在Android Studio工程中使用某一maven库

* 打开项目的根build.gradle文件，声明需要使用的私服地址
```
  allprojects {    
    repositories {        
      jcenter()        
      maven { 
        url 'http://localhost:8081/nexus/content/repositories/android-lib/' 
      }        
    }
  }
```
* 在对应模块的build.gradle文件中，添加项目依赖
```
compile '{groupId}:{artifactId}:{version}@{packaging}'
```
这里的组成和提交的pom.project的信息有关，比如：
```
  <dependency>
    <groupId>com.vic</groupId>
    <artifactId>myrecyclerview</artifactId>
    <version>1.2.0</version>
    <type>aar</type>
  </dependency>
```

## 关于Gradle缓存
在执行过一次Gradle的同步之后，Gradle会把对应的Library的文件下载在本地，之后会直接使用。所以当我们删除旧的Library，用同样的pom.project信息重新上传一个新的Library时，执行Gradle同步，并不会更新最新的Library下来。这个时候可以到仓库存储路径下把对应的Library文件删除。
一般来说，
Mac系统默认下载到：/Users/(用户名)/.gradle/caches/modules-2/files-2.1
Windows系统默认下载到：C:\Users\(用户名)\.gradle\caches\modules-2\files-2.1



## 参考文献

1 用AndroidStudio发布Libs到Bintray jCenter  
http://www.cnblogs.com/jacksBlogs/p/5622948.html

2 巧用 GitHub 创建自己的私人 Maven 仓库，及一些开发Library的建议  
http://www.jianshu.com/p/d2fae8c7d93f

3 使用 Android Studio ＋ Nexus 搭建 Maven 私服 
http://www.jianshu.com/p/f815a05c627e  
http://www.jianshu.com/p/a8aac4d95214

4 Nexus Repository Manager 3.2 Documentation
http://books.sonatype.com/nexus-book/reference3/install.html#service-windows

5 Android Studio发布项目到Maven仓库
http://blog.csdn.net/h_zhang/article/details/51558800

6 在Android Studio中发布Library到jCenter公共仓库
https://yangbo.tech/2015/10/19/distribute-android-library-to-jcetner/

7 拥抱 Android Studio 之四：Maven 仓库使用与私有仓库搭建
http://blog.bugtags.com/2016/01/27/embrace-android-studio-maven-deploy/#简介

