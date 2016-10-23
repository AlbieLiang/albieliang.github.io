---
layout: post
title: ArbitraryGen之脚本模式（使用）
tagline:
category: tools
tags : [tools, android, java, arbitrarygen, code generator, script mode, template, script engine]
---
{% include JB/setup %}
本文章将介绍代码生成器ArbitraryGen的脚本模式（script mode）的使用方法。如果你还不是很了解什么是ArbitraryGen，请移步ArbitraryGen首页，上面会有ArbitraryGen的简要介绍。

（[点击到ArbitraryGen首页]({{ site.baseurl }}/tools/2016/07/22/arbitrarygen-home-page)）


#### 下面就进入本文的主题了：ArbitraryGen的脚本模式的使用

首先我们需要[引入ArbitraryGen]({{ site.baseurl }}/tools/2016/07/22/arbitrarygen-home-page#user-content-引入ArbitraryGen)到项目当中，详细操作可以移步ArbitraryGen首页，在此不再冗述。


## 下面说下参数配置

在本文中我们将只讲述Gradle下的参数配置，至于Eclipse下的Ant，可以参考项目中的arbitrarygen.xml文件，其它编译脚本可以参考实现。（其本质就是带上一堆的参数来调用ArbitraryGen的可执行jar文件）

    arbitraryGen {
        ...
        scriptEngine {
            enable true
            format "e"
        }
    }

#### 上述配置详解：
* scriptEngine.enable配置是否开启脚本模式
* scriptEngine.format用于配置使用脚本引擎解析处理的文件后缀，格式参照普通模式的配置
* 其它配置项请[查看首页]({{ site.baseurl }}/tools/2016/07/22/arbitrarygen-home-page#引入ArbitraryGen)；


## 数据源

目前脚本模式（script mode）下数据源只支持XML格式的数据，该脚本模式下的Parser还不支持扩展，后续再不上实现。
下面通过一个示例加以说明：（events.e文件，可以到[demo项目](https://github.com/{{ site.author.github }}/ArbitraryGen))中查看）
    <?xml version="1.0" encoding="UTF-8"?>
    <Events
        package="com.albie.test.e"
        delegateTag="events"
        tag="event">

        <event name="DefaultEvent" >
            <field name="name" type="String" default="null"/>
        </event>

        <event name="TestEvent"  final="true" parent="DefaultTestEvent">
            <field name="name" type="String" default="null"/>
            <field name="type" type="int" default="0"/>
            <data static="true" final="true">
                <field name="errCode" type="int"/>
            	  <field name="errType" type="int" default="0"/>
            </data>
        </event>

        <event name="XmlThirdEvent">
            <data static="true" final="true">
                <field name="errCode" type="int"/>
            	  <field name="errType" type="int" default="0"/>
            </data>
        </event>
        <event name="XmlFourthEvent">
            <data>
                <field name="errCode" type="int">
                    <constant name="ERR_CODE_NORMAL" value="0"/>
                </field>
            	<field name="errType" type="int" default="0">
                    <constant name="ERR_TYPE_NORMAL" value="0"/>
                    <constant name="ERR_TYPE_NET" value="1"/>
                </field>
            </data>
            <CallBack>
                <field name="trigger" type="String"/>
            </CallBack>
            <result>
                <field name="result" type="String"/>
            </result>
        </event>
    </Events>

如果你看过[ArbitraryGen的普通模式]({{ site.baseurl }}/tools/2016/07/30/arbitrarygen-general-mode-instruction#数据源)，是不是觉得上面的数据源很熟悉呢？是的，用的几乎是一样的数据源，详细可以对比一下，稍微有些不同点。

上面的数据源文件中有几个小点需要特别说一下：
* 根节点的package属性，是用来指定其一级子节点的对应的类的包名
* 根节点的tag属性，是用于指定其一级节点对应的所有需要生成代码的节点类型，并且与生成代码的模板存在[映射关系](#模板映射关系)，tag标签里面是一个有逗号“,”分隔的列表，如果tag属性不存在，则会直接读取根节点的一级节点名列表作为tag的值
* 根节点的delegateTag属性，用于指定什么这个资源文件对应的委派类的生成代码模板对应的tag（委派类，可能会让大家感觉很难理解，可以简单理解，就是用于收集该数据源文件对应所有生成的类的一个收集类）
* 数据源文件中的每个单元数据都将被转成JSON格式的数据作为代码模板引擎的输入数据，其命名规则为：（1）子节点：可直接通过节点引用，如data.field;(2)属性：通过属性key前面加个下划线“_”来引用，如field._name;

## 代码模板

下面我先贴上生成代码的模板（文件名为event.vigor-template，[也可以到demo项目中查看](https://github.com/{{ site.author.github }}/ArbitraryGen)）：

{% raw %}

    package <%=_package%>;

    import cc.suitalk.event.rx.RxEvent;

    public final class <%=_name%> extends RxEvent {
    	public final static String ID = "AutoGenEvent.<%=_name%>";

    	<% for (var i = 0; this.data && i< data.length; i++) { var field = data[i];%>
    		<% if (field.constant) { %>
    			<% if (field.constant.length) { %>
    			<% for (var k = 0; k < field.constant.length; k++) {var constant = field.constant[k];%>
    				public final static <%=field._type%> <%=constant._name%> = <%=constant._value%>;
    			<%}%>
    			<%} else {%>
    				public final static <%=field._type%> <%=field.constant._name%> = <%=field.constant._value%>;
    			<%}%>
    		<%}%>
    	<%}%>
    	<% for (var i = 0; this.result && i < result.length; i++) { var field = result[i];%>
    		<% if (field.constant) {%>
    			<% if (field.constant.length) {%>
    			<% for (var k = 0; k < field.constant.length; k++) {var constant = field.constant[k];%>
    				public final static <%=field._type%> <%=constant._name%> = <%=constant._value%>;
    			<%}%>
    			<%} else {%>
    				public final static <%=field._type%> <%=field.constant._name%> = <%=field.constant._value%>;
    			<%}%>
    		<%}%>
    	<%}%>

    	<% if (this.data) {%>
    		<% for (var i = 0; this.data && i < data.length; i++) { var field = data[i];%>
                <% if (field.constant) { %>
                    /**
                     * The value can be :
                     *
                    <% if (field.constant.length) { %>
                        <% for (var k = 0; k < field.constant.length; k++) {var constant = field.constant[k];%>
                            * {@link #<%=constant._name%>};
                        <%}%>
                    <%} else {%>
                        * {@link #<%=field.constant._name%>};
                    <%}%>
                    */
                <%}%>
    			public <%=field._type%> <%=field._name%><% if (field._default) {%> = <%=field._default%><%}%>;
    		<%}%>
    	<%}%>

    	public <%=_name%>() {
    	    this.action = ID;
    	}
    	public <%=_name%>(Callback callback) {
    		this.action = ID;
    		this.callback = callback;
    	}

    	<% if (this.result) {%>
    	public Result result = new Result();
    	public final static class Result {
    		<% for (var i = 0; i < result.length; i++) { var field = result[i];%>
                <% if (field.constant) { %>
                    /**
                     * The value can be :
                     *
                    <% if (field.constant.length) { %>
                        <% for (var k = 0; k < field.constant.length; k++) {var constant = field.constant[k];%>
                            * {@link #<%=constant._name%>};
                        <%}%>
                    <%} else {%>
                        * {@link #<%=field.constant._name%>};
                    <%}%>
                    */
                <%}%>
    			public <%=field._type%> <%=field._name%><% if (field._default) {%> = <%=field._default%><%}%>;
    		<%}%>
    	}
    	<%}%>
    }

{% endraw %}

如果你学过JSP，是不是看到这个模板代码是不是有点熟悉呢？恩，没错，模板文件中的用于区分脚本代码的标签就是<%%>，这个跟JSP中的标签还是比较像的，但是模板文件中采用的是JavaScript脚本，目前ArbitraryGen的脚本模式下只支持JavaScript脚本格式，后期将会加上可扩展接口，以支持更多的脚本语言。

在模板文件中脚本所用的的所有数据均来自原[数据源](#数据源)文件，其命名规则上面已经提到，在此不再多说了。

## 模板映射关系

看到上面你可能会疑惑，这个模板是怎么与数据源对应上的呢？这里就是我们要讲的模板关系映射文件template-mapping.properties了，看到是.properties文件，应该就不用我多说之中的格式了吧！直接贴上文件的内容吧。（没接触过properties格式文件的同学可自行搜索学习哈~）

template-mapping.properties文件内容如下：
{% raw %}

    # The relationship mapping for template and tag
    #
    # Just read hybrids generate code engine to know more.
    #
    table=VigorDBItem.vigor-template
    hybrid-table=VigorDBItem.hybrid-template
    event=event.vigor-template

{% endraw %}

其中event这个key值对应着[数据源](#数据源)文件中的tag标签中的值（当然，tag标签里面是一个有逗号“,”分隔的列表），如果tag属性不存在，则会直接读取根节点的一级节点名列表作为tag的值。

## 生成出来的代码

生成了如下几个文件：

![arbirarygen-script-mode-output-code]({{ ASSET_PATH }}/images/arbirarygen-script-mode-output-code.png)

生成的代码我就补贴了，如果你跑了demo程序，并按照上述说的步骤操作了，你会发现events.e定义的数据，并不是所有节点都被输出了，如果想知道为什么，就移步到[代码模板](#代码模板)中详细看看脚本的逻辑，其实脚本逻辑还是比较容易看懂的。如有什么问题可以交流，本文到此结束了~
