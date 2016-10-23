---
layout: post
title: ArbitraryGen，不只是一个代码生成器
tagline:
category: tools
tags : [tools, android, java, arbitrarygen, code generator]
---
{% include JB/setup %}
ArbitraryGen是一套代码生成工具，同时也是一套代码生成框架，它提供了基本的代码生成能力，同时也提供了可扩展能力，可以自定义数据源格式（Source File）和解析器（Parser），可以自定义源数据转换器（Convertor），同时还可以实现SDK中的Wrapper接口来对中间的数据的处理。
ArbitraryGen处理提供上面所提到的基本代码生成能力之外，还可以支持脚本模板生成代码的模式，即：代码生成的模板是一个脚本文件，而不用通过Java代码来输出一系列的字符串，还有一个特色的功能就是，模板可以是普通可编辑的源码（如普通的*.java文件），这个就是所谓的混合模式。
ArbitraryGen还提供了一个比较鸡肋的功能，当时年少不懂事，为了实现一个功能，做起了解析java文件的语法分析器（应该叫半语法分析器），也就是可以在非编译环境下把*.java文件当做数据源文件，加以解析，并提供给ArbitraryGen这个代码生成器使用。（可以说是吃力不讨好，这个可以需要太多的精力投入了）

该项目github地址为[https://github.com/{{ site.author.github }}/ArbitraryGen](https://github.com/{{ site.author.github }}/ArbitraryGen)

### 支持一下几种模式

#### 1、基本模式
默认支持XML格式的数据源解析（可自行扩展），并输出支持Java代码。

[使用教程]({{ site.baseurl }}/tools/2016/07/30/arbitrarygen-general-mode-instruction)
[设计实现]({{ site.baseurl }}/tools/2016/08/01/arbitrarygen-general-mode-design)

#### 2、脚本模式
默认支持XML格式的数据源（可自定义实现其它数据源的支持），并按照脚本模板输出相应格式的代码。（不局限于Java代码）

[使用教程]({{ site.baseurl }}/tools/2016/07/30/arbitrarygen-script-mode-instruction)
[设计实现]({{ site.baseurl }}/tools/2016/08/07/arbitrarygen-script-mode-design)

#### 3、混合模式
默认支持XML格式的数据源（可自定义实现其它数据源的支持），可在源文件中生成代码片段，同时不影响源代码的逻辑。（不局限于Java代码）

[使用教程]({{ site.baseurl }}/tools/2016/07/23/arbitrarygen-hybrid-mode-instruction)
[设计实现]({{ site.baseurl }}/tools/2016/08/17/arbitrarygen-hybrid-mode-design)

#### 4、自定义代码解析器模式
改模式目前只支持Java语言，是鸡肋，不多说了。

[使用教程]({{ site.baseurl }}/tools/2016/08/30/arbitrarygen-custom-parser-mode-instruction)
[设计实现]({{ site.baseurl }}/tools/2016/08/30/arbitrarygen-custom-parser-mode-design)

### 引入ArbitraryGen

#### 1）在Android Studio中引入
其实严格来说就是支持gradle 构建的工程中引入的方法如下：

##### （1）gradle插件方式

在project下的build.gradle文件中引入插件：

    buildscript {
        repositories {
            jcenter()
            maven {
                url 'https://dl.bintray.com/albieliang/maven/'
            }
        }

        dependencies {
            classpath 'com.android.tools.build:gradle:2.1.2'
            classpath 'cc.suitalk.tools:arbitrarygen-plugin:1.0.0'
        }
    }
    allprojects {
        repositories {
            jcenter()
            maven {
                url 'https://dl.bintray.com/albieliang/maven/'
            }
        }
    }
在module下配置ArbitraryGen参数：

    apply plugin: 'com.android.library'
    apply plugin: 'arbitrarygen' //引入插件
    android {
        ...
    }
    dependencies {
        ...
    }
    // 参数配置
    arbitraryGen {
        libsDir "${project.rootDir.getAbsolutePath()}/ArbitraryGen" // 指定代码生成器资源库根目录，其中包含生成代码的模板，模板映射文件和代码生成器可执行文件等
        inputDir "../$name/autogen" // 指定数据源文件目录
        outputDir "$buildDir/generated/source/autogen" // 指定“普通”（相对于“混合式”）代码生成的目标目录
        // 普通设置相关的参数，下面只是列出一少部分，更多参数将在后续不上说明
        general {
            format "xml,event,db" // 指定数据源文件的后缀列表（非“混合式”代码生成器模块）
        }
        // log相关的参数，下面只是列出一少部分，更多参数将在后续不上说明
        logger {
            debug true
            logToFile true
            printArgs true
            path "$buildDir/outputs/logs/ag.log" // 指定log输出目录
        }
    }

上述配置详解：

* libsDir用于配置ArbitraryGen资源库的根目录，其中会包含可执行文件ArbitraryGen.jar、必要的脚本库目录core-libs和模板库目录template-libs等
* inputDir用于配置ArbitraryGen数据源根目录，就是提供数据用于生成代码的源数据文件存放的目录
* outputDir用于配置ArbitraryGen代码生成输出的根目录
* general.format用于配置使用默认源数据解析器解析（默认解析器支持格式为XML）的文件后缀
* general.extParsers用于配置自定义的Parser（解析器）、Convertor（转换器）和Wrapper（用于对生成代码过程生成的中间数据进行处理，所谓“包装”）等
* logger.debug，若为true则log中将会输出所有等级的log
* logger.logToFile用于配置是否将log输出到文件当中
* logger.printArgs，若为true则输出，调用ArbitraryGen.jar时输入的所有参数
* logger.path用于配置log输出的文件（包含路径）

其它具体某种模式的配置选项将在具体模式的使用介绍中讲述。

想了解更多的配置信息可以查看该[项目源码](https://github.com/{{ site.author.github }}/ArbitraryGen)

###### （2）直接使用gradle脚本调用方式
这个其实也很简单，就是写一个运行可执行文件的命令而已，可以参考该项目中gradle插件的实现逻辑（项目中的arbitrarygen-plugin module里面的ArbitraryGenTask.grovvy文件），在这里我就暂时不讲，有需要的同学可以找我。

##### 2）在Eclipse中引入
我在项目的ArbitraryGen/usage-chatter目录下放了两个脚本文件，其中一个是ant-script.xml文件，该文件是可以在Eclipse中使用Ant来编译时使用ArbitraryGen，它将在编译前完成代码的生成。

##### 3）其它场景中的引入及使用
可以参考上面描述的编写gradle或Ant脚本的方式，做相应的调整。

### 待完善功能
+ 存储数据源文件modified time，优化代码生成频次
+ 混合模式下，生成的代码会出现较多的换行问题；（这个问题后续再搞，只是看起来没那么好看而已，不影响使用）
+ 脚本标签<%script code%>支持换行；（目前不支持）
+ java源码解析器，对方法参数带有Annotation这种case没有处理；
+ Gradle插件还未发布到公共的maven仓库上；

### 涉及到的相关知识
* [自定义Gradle插件]({{ site.baseurl }}/gradle/2016/08/30/custom-gradle-plugin)
* [XML格式数据的解析](#)
* [捕捉Java异常](#)
* [Gradle打包]({{ site.baseurl }}/lessons/2016/09/07/pack-jar-by-gradle)
* [Maven仓库的使用](#)
