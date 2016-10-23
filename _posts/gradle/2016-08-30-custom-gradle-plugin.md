---
layout: post
title: 自定义Gradle插件
tagline:
category: gradle
tags : [gradle plugin, groovy, android, arbitrarygen]
---
{% include JB/setup %}
本文将介绍如何编写一个自定义的Gradle Plugin，我编写的第一个Gradle Plugin是为了给我开发的一个[代码生成器ArbitraryGen]({{ site.baseurl }}/arbitrarygen-home-page)所使用的，所以本文全过程将是讲述如何编写ArbitraryGen的Gradle Plugin。

下面开始我们正文了：

## 创建Groovy工程

由于Gradle是采用Groovy作为编程语言，所以我们在编写插件的时候用的也是groovy，我并没有系统的学过groovy，但是我们有google和百度，所以很多问题都可以解决，有groovy不懂的语法什么的都可以通过搜索来解决，所以这个不是问题哈~

如果你平时使用Android Studio开发，你会发现AS并没有方法直接创建一个Groovy工程，所以我们就通过手动的方式来创建一个Groovy工程，然后再通过AS来编辑（当然也可以直接用文本编辑器和终端来完成整个编写和编译过程）。

#### 项目目录结构

![arbitrarygen-gradle-plugin]({{ site.baseurl }}/images/arbitrarygen-gradle-plugin.png)

首先我们创建一个空的目录ArbitraryGen作为工程的根目录，在该目录下创建一个空的目录arbitrarygen-plugin作为插件Module（当然这里也可以直接一个project目录就够了，因为我的项目当中不知一个Gradle Plugin的Module，所以Plugin工程是以module的形式存在的，而且在后面我们还要创建一个demo module来演示这个插件的使用）。

#### 编译脚本

接下来我们需要分别在ArbitraryGen和arbitrarygen-plugin各创建一个空的build.gradle文件

更目录下的build.gradle文件就是一个普通的Android Project的build.gradle文件（暂时不用添加什么特殊的配置，直接copy过来使用即可）
该文件内容可以为：

    buildscript {
        repositories {
            jcenter()
        }
        dependencies {
            classpath 'com.android.tools.build:gradle:2.1.2'
        }
    }
    allprojects {
        repositories {
            jcenter()
        }
    }

    task clean(type: Delete) {
        delete rootProject.buildDir
    }


插件Module arbitrarygen-plugin根目录目录下build.gradle文件如下：

    apply plugin: 'groovy'
    apply plugin: 'maven'

    // Maven Group ID for the artifact
    group = "cc.suitalk.tools"

    // 发布的资源对应的版本号
    version = '1.0.0'
    // 发布资源的项目名
    archivesBaseName = 'arbitrarygen'

    repositories {
        mavenCentral()
    }

    dependencies {
        compile gradleApi()
        compile localGroovy()
        compile files('libs/proguard.jar')
    }

    // 一定要记得使用交叉编译选项，因为我们可能用很高的JDK版本编译，为了让安装了低版本的同学能用上我们写的插件，必须设定source和target
    compileGroovy {
        sourceCompatibility = 1.7
        targetCompatibility = 1.7
        options.encoding = "UTF-8"
    }

    uploadArchives {
        repositories.mavenDeployer {
            // 如果你公司或者自己搭了nexus私服，那么可以将插件deploy到上面去
            //        repository(url: "http://XXX.XXX.XXX.XXX:8080/nexus/content/repositories/releases/") {
            //            authentication(userName: "admin", password: "admin")
            //        }
            // 目前我们先把它发布到本地，
            // 上面的注释已经提供发布到nexus私服的方法，
            // 后面我们将介绍发布到公共maven库的方法
            repository(url: "file:../repositories/release")
        }
    }

上述的group、archivesBaseName和version三个条件能唯一的确认maven仓库中的资源（当然maven仓库有URL来定位）。

前面的arbitrarygen-plugin目录图中，可以看出，我们需要一个存放插件源码的目录src/main/groovy和存放插件信息的src/main/resources目录。

#### 插件实现

现在我们要实现一个Gradle插件了，首先在src/main/groovy下创建一个package为cc.suitalk.gradle.plugin，接下来我们新建一个名叫ArbitraryGenPlugin实现了Plugin<Project>接口的class，在apply方法中实现自己的逻辑即可。

下面是ArbitraryGen插件实现逻辑：

    package cc.suitalk.gradle.plugin

    import org.gradle.api.Plugin
    import org.gradle.api.Project

    /**
     *
     * @Author AlbieLiang
     *
     */
    class ArbitraryGenPlugin implements Plugin<Project> {

        Project project;

        ArbitraryGenPluginExtension extension;

        LoggerArgs loggerExtension;
        GeneralArgs argsExtension;
        ScriptEngineArgs scriptEngineExtension;

        ArbitraryGenTask arbitraryGenTask;

        @Override
        void apply(Project project) {
            this.project = project
            this.extension = project.extensions.create("arbitraryGen", ArbitraryGenPluginExtension)
            this.loggerExtension = this.extension.extensions.create("logger", LoggerArgs)
            this.argsExtension = this.extension.extensions.create("general", GeneralArgs)
            this.scriptEngineExtension = this.extension.extensions.create("scriptEngine", ScriptEngineArgs)

            this.arbitraryGenTask = this.project.tasks.create("arbitraryGen", ArbitraryGenTask)

            this.project.tasks.whenTaskAdded { task ->
                this.project.getLogger().debug("when task(${task.name}) added in project(${project.name}).")
                if (task.name.startsWith('generate') && task.name.endsWith('Sources')) {
                    println("add task(${task.name}) project : ${this.project}, name : $name.")
                    task.dependsOn arbitraryGenTask
                }
            }

            project.afterEvaluate {
                println "arbitrarygen(${this.project.name}) libsDir : '${this.extension.libsDir}'"
                println "arbitrarygen(${this.project.name}) inputDir : '${this.extension.inputDir}'"
                println "arbitrarygen(${this.project.name}) outputDir : '${this.extension.outputDir}'"

                arbitraryGenTask.inputDir = this.project.file(this.extension.inputDir)
                arbitraryGenTask.outputDir = this.project.file(this.extension.outputDir)
                arbitraryGenTask.libsDir = this.project.file(this.extension.libsDir)

                arbitraryGenTask.loggerArgs = this.loggerExtension
                arbitraryGenTask.generalArgs = this.argsExtension
                arbitraryGenTask.scriptEngineArgs = this.scriptEngineExtension

                if (arbitraryGenTask.inputDir == null || !arbitraryGenTask.inputDir.exists()) {
                    println("project: ${this.project} do not exists arbitrarygen dir.")
                    return
                }
            }
        }
    }

如果你仔细看上面的代码，你应该会注意到获取插件外部的输入是通过this.extension = project.extensions.create("arbitraryGen", ArbitraryGenPluginExtension)这句话得到的，如果你使用过ArbitraryGen的话，你应该会注意到"arbitraryGen"参数正是ArbitraryGen参数extension，其实这个extension就是普通的一个class，其中包含了需要配置的参数的定义，上述的ArbitraryGenPluginExtension定义如下：

    package cc.suitalk.gradle.plugin;

    class ArbitraryGenPluginExtension {

        String libsDir;
        String inputDir;
        String outputDir;

        boolean enable;

        public ArbitraryGenPluginExtension() {
        }
    }

在build.gradle文件中的使用（非apply）如下：

    arbitraryGen {
        libsDir "${project.rootDir.getAbsolutePath()}/ArbitraryGen"
        inputDir "${project.projectDir.absolutePath}/autogen"
        outputDir "$buildDir/generated/source/autogen"
    }

其它的可以参考上面来自行解决问题。插件已经实现了，参数输入也定义了，接下来我们需要配置插件的相关信息了。

#### 插件配置

这时候我们需要src/main/resources目录下创建一个两级目录META-INF/gradle-plugins，在gradle-pugins目录下新建一个properties文件，文件名为插件被引用时使用的名字，如ArbitraryGen中使用到的就是arbitrarygen，所以该文件名为arbitrarygen.properties。
在该文件中配置插件类，文件内容如下如：

    implementation-class=cc.suitalk.gradle.plugin.ArbitraryGenPlugin

这样我们引用插件的时候就有理有据了，哈哈~（其实类似于打包可运行的jar包一样，需要一个入口的配置）

接下来我们只需要通过gradle build把插件打包成jar包（在build/libs目录下），然后把jar包拷贝到相应项目中，可以直接使用啦~
用法跟普通的插件一样apply一下就好（记得名字是跟META-INF/gradle-plugins目录下那个properties文件名一致），这里就不多说了哈~（或者你可以参考[ArbitraryGen这个项目](https://github.com/{{ site.author.github }}/ArbitraryGen)）

如果你想将插件发布到本地maven仓库或某个目录下，可以回头看下上面的脚本，运行一下gradle uploadArchives命令即可。（具体的路径可以自行修改）

至于如何把插件发布到公共maven仓库上面，将会在另外一篇文章中介绍~
