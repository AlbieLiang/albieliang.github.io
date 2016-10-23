---
layout: post
title: ArbitraryGen之普通模式（使用）
tagline:
category: tools
tags : [tools, android, java, arbitrarygen, code generator, general mode, template]
---
{% include JB/setup %}
本文章将介绍代码生成器ArbitraryGen的普通模式（general mode）的使用方法。如果你还不是很了解什么是ArbitraryGen，请移步ArbitraryGen首页，上面会有ArbitraryGen的简要介绍。

（[点击到ArbitraryGen首页]({{ site.baseurl }}/tools/2016/07/22/arbitrarygen-home-page)）


## 下面就进入本文的主题了：ArbitraryGen的普通模式的使用

首先我们需要[引入ArbitraryGen]({{ site.baseurl }}/tools/2016/07/22/arbitrarygen-home-page#user-content-引入ArbitraryGen)到项目当中，详细操作可以移步ArbitraryGen首页，在此不再冗述。


## 下面说下参数配置

在本文中我们将只讲述Gradle下的参数配置，至于Eclipse下的Ant，可以参考项目中的arbitrarygen.xml文件，其它编译脚本可以参考实现。（其本质就是带上一堆的参数来调用ArbitraryGen的可执行jar文件）

    arbitraryGen {
        libsDir "${project.rootDir.getAbsolutePath()}/ArbitraryGen"
        inputDir "${project.projectDir.absolutePath}/autogen"
        outputDir "$buildDir/generated/source/autogen"

        general {
            format "event,db" // Use XML format parser.
            extParsers = ["../ArbitraryGen/arbitrarygen-demo-wrapper.jar,cc.suitalk.tools.arbitrarygen.demo.ExternalTemplateConvertor", "../ArbitraryGen/arbitrarygen-demo-wrapper.jar,cc.suitalk.tools.arbitrarygen.demo.EventTemplateWrapper", "../ArbitraryGen/arbitrarygen-demo-wrapper.jar,cc.suitalk.tools.arbitrarygen.demo.ExternalTemplateWrapper", "../ArbitraryGen/arbitrarygen-demo-wrapper.jar,cc.suitalk.tools.arbitrarygen.demo.greendao.GreenDaoConvertor"]
        }

        logger {
            debug true
            logToFile true
            printArgs true
            printTag true
            path "$buildDir/outputs/logs/ag.log"
        }
    }

#### 上述配置详解：
* libsDir用于配置ArbitraryGen资源库的根目录，其中会包含可执行文件ArbitraryGen.jar、必要的脚本库目录core-libs和模板库目录template-libs等
* inputDir用于配置ArbitraryGen数据源根目录，就是提供数据用于生成代码的源数据文件存放的目录
* outputDir用于配置ArbitraryGen代码生成输出的根目录
* general.format用于配置使用默认源数据解析器解析（默认解析器支持格式为XML）的文件后缀
* general.extParsers用于配置自定义的Parser（解析器）、Convertor（转换器）和Wrapper（用于对生成代码过程生成的中间数据进行处理，所谓“包装”）等；
* logger用于配置ArbitraryGen log输出相关的参数，[详细请看首页]({{ site.baseurl }}/tools/2016/07/22/arbitrarygen-home-page#引入ArbitraryGen)；

## 数据源

在上述inputDir所配置的目录下添加相应的数据源文件，下面我们使用event为后缀的XML格式的数据源文件作为示例：
![general-mode-demo-src]({{ ASSET_PATH }}/images/arbitrarygen-general-mode-demo-src.png)

图中的events.event文件具体内容如下：（[也可以到demo项目中查看](https://github.com/{{ site.author.github }}/ArbitraryGen)）

    <?xml version="1.0" encoding="UTF-8"?>
    <Events package="com.albie.test.event">

        <event name="Default" >
            <field name="name" type="String" default="null"/>
        </event>

        <event name="Test"  final="true" parent="DefaultTestEvent">
            <field name="name" type="String" default="null"/>
            <field name="type" type="int" default="0"/>
            <Data static="true" final="true">
                <field name="errCode" type="int"/>
            	<field name="errType" type="int" default="0"/>
            </Data>
        </event>

        <event name="XmlThird">
            <Data static="true" final="true">
                <field name="errCode" type="int"/>
            	<field name="errType" type="int" default="0"/>
            </Data>
        </event>
        <event name="XmlFourth">
            <Data>
                <field name="errCode" type="int"/>
            	<field name="errType" type="int" default="0"/>
            </Data>
            <CallBack>
                <field name="trigger" type="String"/>
            </CallBack>
            <Result>
                <field name="result" type="String"/>
            </Result>
        </event>
    </Events>

#### 数据源文件的简单介绍
* 根节点中的package属性值指定其一级子节点生成的代码原文件存储的包名（与outputDir共同决定文件存放的目录）
* 根节点的每一个一级节点都对应着一个代码生成对象（即可能对应着一个被生成的文件），至于每个不同类型的节点类型可以使用不同的Parser、Convertor和Wrapper对其进行处理，[将在后面介绍](#自定义Parser、Convertor和Wrapper)，该节点定义了一个类的基本信息，其属性包括：name（指定域的名字，若不存在改属性则使用节点名作为域的名字）、type（指定域的数据类型）、default（指定默认值）、static、final、modifier（可为public、private、protected或空）、interfaces（指定实现的接口）、parent（指定继承的父类）
* 根节点的每一个一级节点的子节点中，field节点对应类中的属性（域），field标签中目前可用的属性有name（指定域的名字）、type（指定域的数据类型）、default（指定默认值）、static、final、modifier（可为public、private、protected或空）、genGetter（可为true或false）、genSetter（可为true或false）等
* 根节点的每个一级节点中的子节点中，非field节点都对应着一个类的定义，其设计是根节点的一级节点的一个递归概念，除此之外，该类会在其父节点对应的类中创建一个该类的域（变量）

## 自定义Parser、Convertor和Wrapper

想必看了上面的使用介绍之后，你对ArbitraryGen的普通模式有所了解了吧！ArbitraryGen提供的上述能力，只能满足一小部分需求，如果我们需要做比较复杂的代码生成过程怎么办呢？下面我们就开始讲一下如何定制化。

在定制化之前，需要新建一个Java工程，然后引入arbitrarygen-sdk.jar到libs中，这样就可以开始开发了。

#### 自定义Parser
Parser的出现是为了支持更多的数据格式，由于ArbitraryGen默认支持的格式只有XML一种，有一定的局限性，所以提供了一个自定义Parser的接口，以让ArbitraryGen更具扩展性及更具想象力。
实现一个接口ICustomizeParser，并实现canParse和parse方法即可，其中canParse接口用于判决该Parser是否能够解析suffix后缀的数据源文件，而parse方法则是解析的逻辑实现，并返回解析产物RawTemplate（[详细可以看源码](https://github.com/{{ site.author.github }}/ArbitraryGen)），下面为示例，可以作为参考。

![general-mode-demo-custom-parser]({{ ASSET_PATH }}/images/arbirarygen-general-mode-custom-parser.png)

接口实现如下：

    package cc.suitalk.tools.arbitrarygen.demo;

    import java.io.File;
    import java.util.List;

    import cc.suitalk.arbitrarygen.extension.ICustomizeParser;
    import cc.suitalk.arbitrarygen.template.RawTemplate;
    import cc.suitalk.arbitrarygen.utils.Log;
    import cc.suitalk.arbitrarygen.utils.XmlUtils;

    public class ExternalTemplateParser  implements ICustomizeParser {

    	private static final String TAG = "AG.demo.ExternalTemplateParser";

    	@Override
    	public boolean canParse(String suffix) {
    		return "ext".equalsIgnoreCase(suffix);
    	}

    	@Override
    	public List<RawTemplate> parse(File file) {
    		if (file == null || !file.exists() || !file.isFile()) {
    			Log.e(TAG, "The file is null or do not exists or isn't a file.");
    			return null;
    		}
    		return XmlUtils.getXmlNotes(file);
    	}

    }
上述的示例中使用的还是XML格式的解析器，如果需要解析其它格式的，可以自行添加相应的解析逻辑，在此不再冗述。
既然我们已经实现了一个自定义的Parser了，那么现在就要使用了。
首先，我们需要编译一个lib jar包（可以参考Gradle编译lib jar包），注意这里的jar包，不能包含上面引用的arbitrarygen-sdk.jar，否则会出现类重定义的crash（因为ArbitraryGen.jar中已经包含了arbitrarygen-sdk.jar中的代码了）。
接下来我们需要在项目中引入自定义的Parser了，记忆力好的同学应该会记得上面的“[下面说下参数配置](#下面说下参数配置)”中已经提到过了，其实很简单，我们就在general.extParsers中添加我们自定Parser的jar文件位置和实现的类名（包括包信息）即可，如下：

    arbitraryGen {
        ...
        general {
            format "ext"
            extParsers = ["../ArbitraryGen/arbitrarygen-demo-wrapper.jar,cc.suitalk.tools.arbitrarygen.demo.ExternalTemplateParser"]
        }
    }

general.extParsers实际上是一个一维数组，其中的当个数据是又一个逗号“,”分隔，逗号前面的是jar文件的路径，逗号后面是自定义Parser的类名（包括包名在内）。
在这里还有一个很重要的配置需要加上，那就是general.format了，需要添加Parser解析的后缀名，ArbitraryGen才会把该后缀名的文件当做扫描对象，要不Parser就没什么用了哈！

#### 自定义Convertor
上述的Parser是用来解析源数据文件的，而Convertor则是用来将Parser解析得到的RawTemplate对象转换成与类对应的TypeDefineCodeBlock对象，ArbitraryGen中提供了AnalyzerHelper工具类可以直接将RawTemplate原封不动的装换成RawTemplate对象，你可能会想，既然都已经可以达到转换的效果来，为何还要多出那么一个Convertor呢？其实是这样的，RawTemplate是原数据，而TypeDefineCodeBlock是用于输出生成目标代码的对象，Convertor可以作为中间的转换器，从RawTemplate中获取必要信息并加上额外添加的信息用于生成TypeDefineCodeBlock对象。
然而还有另外一种做法，就是生成代码直接在Convertor里面完成，而不是通过ArbitraryGen原本的生成代码流程，这时候只要convert方法返回值是null即可。这种用法，我将在《[如何使用ArbitraryGen生成GreenDao代码]({{ site.baseurl }}/lessons/2016/08/09/arbitrarygen-demo-greendao)》中介绍到。

与自定义Parser类似，自定义Convertor需要实现一个ICustomizeConvertor接口，实现canConvert和convert方法即可。
下面为示例：（[详细可以看源码](https://github.com/{{ site.author.github }}/ArbitraryGen)）

![general-mode-demo-custom-convertor]({{ ASSET_PATH }}/images/arbirarygen-general-mode-custom-convertor.png)

接口实现如下：

    package cc.suitalk.tools.arbitrarygen.demo;

    import java.util.Map;

    import cc.suitalk.arbitrarygen.block.TypeDefineCodeBlock;
    import cc.suitalk.arbitrarygen.core.ContextInfo;
    import cc.suitalk.arbitrarygen.core.TemplateConstants;
    import cc.suitalk.arbitrarygen.extension.ICustomizeConvertor;
    import cc.suitalk.arbitrarygen.template.RawTemplate;
    import cc.suitalk.arbitrarygen.utils.AnalyzerHelper;

    /**
     *
     * @author albieliang
     *
     */
    public class ExternalTemplateConvertor implements ICustomizeConvertor {

    	private static final String TAG_NAME = "Ext";

    	public ExternalTemplateConvertor() {
    	}

    	@Override
    	public TypeDefineCodeBlock convert(ContextInfo contextInfo, RawTemplate rawTemplate) {
    		if (rawTemplate == null) {
    			return null;
    		}
    		doAttachDefaultValues(rawTemplate);
    		return AnalyzerHelper.createTypeDefineCodeBlock(rawTemplate);
    	}

    	private void doAttachDefaultValues(RawTemplate rawTemplate) {
    		Map<String, String> attrs = rawTemplate.getAttributes();
    		String name = attrs.get(TemplateConstants.TEMPLATE_KEYWORDS_NAME);
    		attrs.put(TemplateConstants.TEMPLATE_KEYWORDS_NAME, name + TAG_NAME);
    		attrs.put(TemplateConstants.TEMPLATE_KEYWORDS_PARENT, "Object");
    	}

    	@Override
    	public boolean canConvert(RawTemplate template) {
    		return template != null && TAG_NAME.equalsIgnoreCase(template.getName());
    	}
    }

至于引入项目当中，可以[参考自定义Parser](#自定义Parser)，这里就不在冗述了。


#### 自定义Wrapper

Wrapper这个东西与Parser和Convertor有很大区别，它不是必要的（之所以不用自己定义Parser和Convertor就可以正常使用，是因为ArbitraryGen中已经存在默认Parser和Convertor的实现），而Wrapper是专门抽象出来提供给第三方来处理原数据RawTemplate和中间数据TypeDefineCodeBlock而使用的，它的doWrap方法会在解析得到RawTemplate之后和转换得到TypeDefineCodeBlock之后调用，也就是Wrapper可以把RawTemplate和TypeDefineCodeBlock处理成自己想要的样子，这里就不再多说了，直接给出示例吧。（[详细可以看源码](https://github.com/{{ site.author.github }}/ArbitraryGen)）

![general-mode-demo-custom-wrapper]({{ ASSET_PATH }}/images/arbirarygen-general-mode-custom-wrapper.png)

接口实现如下：

    package cc.suitalk.tools.arbitrarygen.demo;

    import java.util.List;

    import cc.suitalk.arbitrarygen.base.Session;
    import cc.suitalk.arbitrarygen.block.ConstructorMethodCodeBlock;
    import cc.suitalk.arbitrarygen.block.FieldCodeBlock;
    import cc.suitalk.arbitrarygen.block.MethodCodeBlock;
    import cc.suitalk.arbitrarygen.block.TypeDefineCodeBlock;
    import cc.suitalk.arbitrarygen.core.ContextInfo;
    import cc.suitalk.arbitrarygen.core.KeyWords;
    import cc.suitalk.arbitrarygen.core.TemplateConstants;
    import cc.suitalk.arbitrarygen.extension.ITemplateWrapper;
    import cc.suitalk.arbitrarygen.model.DefaultKeyValuePair;
    import cc.suitalk.arbitrarygen.statement.NormalStatement;
    import cc.suitalk.arbitrarygen.template.FastAssignRawTemplate;
    import cc.suitalk.arbitrarygen.template.RawTemplate;
    import cc.suitalk.arbitrarygen.utils.TemplateAttributeHelper;
    import cc.suitalk.arbitrarygen.utils.Util;

    /**
     *
     * @author AlbieLiang
     *
     */
    public class EventTemplateWrapper implements ITemplateWrapper {

    	private static final String TAG = "CodeGen.EventTemplateWrapper";
    	private static final String TAG_NAME = "Event";
    	private Session session;

    	public EventTemplateWrapper() {
    		session = new Session();
    		session.Token = TAG.hashCode();
    	}

    	@Override
    	public boolean doWrap(ContextInfo contextInfo, RawTemplate template) {
    		if (template != null && TAG_NAME.equalsIgnoreCase(template.getName())) {
    			TemplateAttributeHelper.setAttribute(template, TemplateConstants.TEMPLATE_KEYWORDS_PARENT, "IEvent");
    			String name = template.getAttributes().get(TemplateConstants.TEMPLATE_KEYWORDS_NAME);
    			TemplateAttributeHelper.appendAttribute(template, TemplateConstants.TEMPLATE_KEYWORDS_NAME, TAG_NAME);
    			TemplateAttributeHelper.appendAttribute(template, TemplateConstants.TEMPLATE_KEYWORDS_IMPORT,
    					"com.albie.test.sdk.IEvent", KeyWords.V_JAVA_KEYWORDS_SIGN_COMMA);
    			// Add a field
    			FastAssignRawTemplate fart = new FastAssignRawTemplate(KeyWords.V_JAVA_KEYWORDS_PUBLIC, true, true,
    					KeyWords.V_JAVA_KEYWORDS_DATA_BASE_TYPE_STRING, "ID");
    			fart.setAttrDefault("\"" + name + "\"").setName(TemplateConstants.TEMPLATE_KEYWORDS_FIELD);
    			List<RawTemplate> elements = template.getElements();
    			elements.add(fart);
    			// Assign session token.
    			template.Token = session.Token;
    			return true;
    		}
    		return false;
    	}

    	@Override
    	public boolean doWrap(ContextInfo contextInfo, TypeDefineCodeBlock template) {
    		if (template != null && template.Token == session.Token) {//
    			for (int i = 0; i < template.countOfTypeDefCodeBlocks(); i++) {
    				TypeDefineCodeBlock ct = template.getTypeDefCodeBlock(i);
    				if (KeyWords.V_JAVA_KEYWORDS_CLASS.equals(ct.getType().genCode(""))) {
    					FieldCodeBlock field = new FieldCodeBlock();
    					field.setName(Util.changeFirstChatToLower(ct.getName()));
    					field.setType(ct.getName());
    					field.setDefault(KeyWords.V_JAVA_KEYWORDS_NEW + " " + ct.getName().getName() + "()");
    					ct.setIsStatic(true);
    					ct.setIsFinal(true);
    					template.addField(field);
    				}
    			}
    			MethodCodeBlock t = new ConstructorMethodCodeBlock(template, new DefaultKeyValuePair("callback", "ICallback"));
    			t.addStatement(new NormalStatement("this.callback = callback"));
    			template.addMethod(t);

    			t = new ConstructorMethodCodeBlock(template);
    			t.addStatement(new NormalStatement("id = ID"));
    			template.addMethod(t);
    			template.setIsFinal(true);
    			return true;
    		}
    		return false;
    	}
    }

至于引入项目当中，可以[参考自定义Parser](#自定义Parser)，这里就不在冗述了。

到这里为止ArbitraryGen的普通模式的使用基本讲完啦！如需了解其它模式，可以返回[首页]({{ site.baseurl }}/tools/2016/07/22/arbitrarygen-home-page)查阅。
