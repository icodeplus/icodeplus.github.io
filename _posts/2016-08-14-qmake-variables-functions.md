---
layout: post
title:  "qmake .pro文件配置之变量和函数"
date:   2016-08-14 14:20:00 +0800
categories:
- QT
- qmake
comments: true
---

> qmake是QT工程编译管理工具，可以实现跨平台编译管理，通过qmake可以产生针对不同平台的
Makefile文件，实现跨平台编译。

## 1、变量
.pro配置文件中可以自定义变量（类似`DEFINES`，`SOURCES`和`HEADER`），自定义变量实现形式如下：

{% highlight shell %}
MY_VARIABLE = value
{% endhighlight %}

一个已定义变量可以给另一个变量赋值：

{% highlight shell %}
# MY_DEFINES包含DEFINES定义内容
MY_DEFINES = $$DEFINES
# 或者
MY_DEFINES = $${DEFINES}
# $${...}这种形式可以用于两个变量的连接，如下形式
TARGET = myproject_$${TEMPLATE}
{% endhighlight %}

利用`$$(...)`在qmake运行时获取qmake环境变量：

{% highlight shell %}
DESTDIR = $$(PWD)
{% endhighlight %}

利用`$(...)`在qmake生成的Makefile执行时获取qmake环境变量：

{% highlight shell %}
# when the Makefile is processed
DESTDIR = $(PWD)
{% endhighlight %}

` $$[...]`在QT构建时获取QT安装配置变量信息：

{% highlight shell %}
message(Qt version: $$[QT_VERSION])
message(Qt is installed in $$[QT_INSTALL_PREFIX])
message(Qt resources can be found in the following locations:)
message(Documentation: $$[QT_INSTALL_DOCS])
message(Header files: $$[QT_INSTALL_HEADERS])
message(Libraries: $$[QT_INSTALL_LIBS])
message(Binary files (executables): $$[QT_INSTALL_BINS])
message(Plugins: $$[QT_INSTALL_PLUGINS])
message(Data files: $$[QT_INSTALL_DATA])
message(Translation files: $$[QT_INSTALL_TRANSLATIONS])
message(Settings: $$[QT_INSTALL_SETTINGS])
message(Examples: $$[QT_INSTALL_EXAMPLES])
message(Demonstrations: $$[QT_INSTALL_DEMOS])
{% endhighlight %}

## 2、函数

### 变量处理函数

变量处理函数封装逻辑代码块，可以`reutrn`返回值，自定义格式如下：

{% highlight shell %}
defineReplace(functionName){
    #function code
}
# 样例
defineReplace(headersAndSources) {
    variable = $$1
    names = $$eval($$variable)
    headers =
    sources =

    for(name, names) {
        header = $${name}.h
        exists($$header) {
            headers += $$header
        }
        source = $${name}.cpp
        exists($$source) {
            sources += $$source
        }
    }
    return($$headers $$sources)
}
{% endhighlight %}

### 条件函数
条件函数只返回`true`或`false`，此类函数只用于条件分支条件判断，一般使用格式如下：

{% highlight shell %}
# 根据文件是否存在做不同处理
exists(filename) {
    # do something
} else 
{
    # do something
}
{% endhighlight %}

自定义函数格式样例如下：

{% highlight shell %}
defineTest(allFiles) {
    files = $$ARGS

    for(file, files) {
        !exists($$file) {
            return(false)
        }
    }
    return(true)
}
{% endhighlight %}

## 3、qmake内置函数
函数返回值通过前置`$$`获取，例如`VAL = $$basename(FILE)`

* ### packagesExist(packages)
    判断指定的包是否存在

{% highlight shell %}
packagesExist(sqlite3 QtNetwork QtDeclarative) {
    DEFINES += USE_FANCY_UI
}
{% endhighlight %}

* ### basename(variablename)
    返回指定文件路径的基本文件名

{% highlight shell %}
        FILE = /etc/passwd
        FILENAME = $$basename(FILE) #passwd
{% endhighlight %}

* ### CONFIG(config)
    判断`CONFIG`变量配置中是否包含某项配置，`CONFIG`变量配置遵循配置顺序，
    同一配置项最后配置值生效

{% highlight shell %}
CONFIG = debug
CONFIG += release   #此处release生效
#第一个参数是要判断的active的选项，后者是互斥的选项的一个集合
CONFIG(release, debug|release):message(Release build!) #will print
CONFIG(debug, debug|release):message(Debug build!) #no print
{% endhighlight %}

* ### contains(variablename, value)
    判断变量`variablename`中是否包含指定的`value`值

{% highlight shell %}
contains( drivers, network ) {
# drivers contains 'network'
message( "Configuring for network build..." )
HEADERS += network.h
SOURCES += network.cpp
    }
{% endhighlight %}

* ### count(variablename, number)
    判断`variablename`列表项个数是否与`number`相等

{% highlight shell %}
options = $$find(CONFIG, "debug") $$find(CONFIG, "release")
count(options, 2) {
    message(Both release and debug specified.)
}
{% endhighlight %}

* ### dirname(file)
    返回`file`文件路径的目录信息

{% highlight shell %}
FILE = /etc/X11R6/XF86Config
DIRNAME = $$dirname(FILE) #/etc/X11R6
{% endhighlight %}

* ### error(string)
    qmake向用户显示指定的`string`错误信息并退出后续执行，该函数只用于报告无法恢复的错误信息。

{% highlight shell %}
error(An error has occurred in the configuration process.)
{% endhighlight %}

* ### eval(string)
    按qmake语法执行`string`语句并返回`trur`，`string`可以定义新变量和给现有变量赋值

{% highlight shell %}
# {}如不需要可以省略
eval(TARGET = myapp) {
    message($$TARGET)
}
{% endhighlight %}

* ### exists(filename)
    判断`filename`指定文件是否存在，`filename`支持正则表达式可以检查多个文件

{% highlight shell %}
exists( $(QTDIR)/lib/libqt-mt* ) {
    message( "Configuring for multi-threaded Qt..." )
    CONFIG += thread
}
{% endhighlight %}

* ### find(variablename, substr)
    在`variablename`中查找并返回所有满足`substr`的值

{% highlight shell %}
    MY_VAR = one two three four
    $$find(MY_VAR, t.*) # 返回two three
{% endhighlight %}

* ### for(iterate, list)
    for循环，通过`iterate`以次遍历`list`中值

{% highlight shell %}
LIST = 1 2 3
for(a, LIST):exists(file.$${a}):message(I see a file.$${a}!)
{% endhighlight %}

* ### include(filename)
    包含文件，一般用于包含`.pri`文件，包含成功返回`true`失败则返回`false`,包含的文件按顺序执行

{% highlight shell %}
        include(shared.pri)
        OPTIONS = standard custom
        !include(options.pri) {
            message( "No custom build options specified" )
            OPTIONS -= custom
        }
{% endhighlight %}

* ### infile(filename, var, val)
    检查`filename`文件中是否包含`var`变量且变量值包含`val`,如果省略`val`，则只判断文件中是否包含变量声明

{% highlight shell %}
infile(shared.pri, CONFIG, debug)
{% endhighlight %}

* ### isEmpty(variablename)
    判断`variablename`变量是否为空，与`count(variablename, 0)`等价

{% highlight shell %}
isEmpty(CONFIG) {
    CONFIG += qt warn_on debug
}
{% endhighlight %}

* ### join(variablename, glue, before, after)
    利用`glue`拼接`variablename`中列表值，在拼接后的值前加前缀`before`，加后缀`after`

{% highlight shell %}
MY_VAR = one two three four
MY_VAR2 = $$join(MY_VAR, " -L", -L) -Lfive
# MY_VAR2值为'-Lone -Ltwo -Lthree -Lfour -Lfive'
{% endhighlight %}

* ### member(variablename, position)
    返回列表`variablename`中指定`position`位置的值，如果没有找到指定位置的值则返回空字符串,
    如果`position`值省略，则返回0位置的值。

{% highlight shell %}
        MY_LIST = a b c d 
        VAL = $$member(MY_LIST, 1) # VAL值为b
        VAL1 = $$member(MY_LIST) # VAL1值为a
{% endhighlight %}

* ### message(string)
    向控制台打印`string`信息

{% highlight shell %}
message( "This is a message" ) # ""引号可以省略
{% endhighlight %}

* ### prompt(question)
    提示`question`信息，要求用户输入数据并返回输入数据

{% highlight shell %}
VAL = $$prompt("input something")
message($${VAL})
{% endhighlight %}

* ### quote(string)
    与`""`作用等价

* ### replace(string, old_string, new_string)
    把`string`中`old_string`字符串替换成`new_string`字符串

{% highlight shell %}
MESSAGE = This is a tent.
message($$replace(MESSAGE, tent, test)) # print "This is a test."
{% endhighlight %}

* ### sprintf(string, arguments...)
    `string`中使用 %1-%9占位符，`arguments`列表项依次替换占位符，替换后函数返回新字符串

{% highlight shell %}
# 打印 "a b c d"
message($$sprintf("%1 %2 %3 %4", a, b, c, d)
{% endhighlight %}

* ### system(command)
    执行`command`指定的shell命令，shell命令执行成功函数返回0,执行失败返回非0

{% highlight shell %}
system(ls /bin):HAS_BIN=FALSE
{% endhighlight %}

可以直接获取shell命令执行打印信息

{% highlight shell %}
UNAME = $$system(uname -s)
contains( UNAME, [lL]inux ):message( This looks like Linux ($$UNAME) to me )
{% endhighlight %}

* ### unique(variablename)
    去除`variablename`列表中重复的值，每个值保留一份

{% highlight shell %}
ARGS = 1 2 3 2 5 1
ARGS = $$unique(ARGS) #1 2 3 5
{% endhighlight %}

* ### warning(string)
    打印`string`内容，与`message(string)`等价