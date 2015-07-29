
seajs进行模块化开发后，如何对代码进行前端构建，如合并、压缩、版本控制；如何管理通用组件；如何进行静态服务配置，如何更新索引关系文件，如何进行生产环境调试。

本文先从目录结构划分开始，然后介绍构建工具怎么处理，再介绍安装使用说明，最后讲nodeJs命令行工具开发

#目录划分

##怎么合并
方案一
一个组件一个js，随着业务的复杂，页面依赖的组件越来越多，导致一次url访问请求数很多。特别是用户第一次进入页面时，要加载所有的资源，要等几十个资源请求load后，才发ajax请求。虽然二次访问时，组件配合缓存可能会减少页面的流量，但与繁多的请求相比，这种性能上提升几乎为0。（现在状况）

方案二
一个页面一个js，这是另外个极端，不能充分利用浏览器缓存。

方案三
均衡方案，开发时一个组件一个js。上线时，一部分组件合并到页面Js中，将请求数降低到理想值，部分组件不合并、以利用缓存。

这里采用方案三。

这里有关seajs的合并，是页面级的js合并依赖的js；如果许多页面级js,都依赖模块a和模块b，这时候，希望合并模块a和模块b,因为路径问题，这里没涉及。

##划分模块
合并之前，要划分下资源。项目里的js,可按照以下划分 

1. 按照使用调用情况分：必加载组件、绝大地方都使用的组件、只有几个页面使用到的js、一个页面对应的js
2. 按照业务分：基础、标准组件（比如弹出窗，backbone跟业务无关的）、业务组件、页面Js。

综合考虑，我们将结构分成如下，并采用业务命名区分

* 基础模块：所有的页面都需要，弄成一个请求，script标签引入。采用合并或者combo请求，比如jquery json2

* 标准模块：外面引入的插件，自己写的插件，比如backbone，dialog

* 业务模块：少数页面用到的业务Js,比如wifi

* 页面模块：页面对应的Js，比如add2.jsp对应的是add.js的入口。

合并时，页面模块合并了业务模块；标准模块之间不合并，其它页面用到相同标准模块时，读取缓存；基础模块合并后被所有页面加载；标准模块内部各组件分别合并成一个js作为外部接口。

##通用组件
通用组件，也就是上面的标准模块，合并压缩后，通过版本号更新网页缓存，它的构建流程跟页面级的js不一样。页面级js,通过别名引用通用组件。

##目录结构

    --pagekage.json //配置
    --app/   //存放页面
    --src/   //打包前目录
    ------config.json  //索引关系文件
    ------page/   //页面级js
    ------page/share/share.js
    ------page/share/share.css
    ------page/share/imgs/
    ------business    //业务模块
    ------business/wifi/
    ------business/wifi/wifi.js
    ------business/wifi/wifi.css
    ------business/wifi/imgs/   
    ------base/   //基础模块，公共js，非seajs方式的js
    ------base/scripts/jquery.js
    ------base/scripts/reset.css
    ------base/imgs/
    ------widget/   //标准模块，公共组件
    ------widget/dialog
    ------widget/dialog/package.json
    ------widget/dialog/examples
    ------widget/dialog/src
    ------widget/dialog/src/dialog.js
    ------widget/dialog/src/dialog.css
    ------widget/dialog/src/imgs
    
一个网页，引用了基础模块，引用了一个页面级的js，页面级的js再引用了业务js，和组件级js。页面级的js放在page下；业务级的js放在business下，将被page合并；公共组件级别的js放在widget下，不会被合并，结构如上所示。每个功能文件夹下放着js、css、imgs。

#怎么处理

处理前文件都在src下，处理后的文件在asset目录下

##page目录
    /*合并压缩前*/
    --src/   //打包前目录
    ------config.json  //索引文件
    ------page/   //页面级js
    ------page/share/share.js
    ------page/share/share.css
    ------page/share/imgs/
    ------page/share/share.jpg
    ------business    //业务模块
    ------business/wifi/
    ------business/wifi/wifi.js
    ------business/wifi/wifi.css
    ------business/wifi/imgs/
    ------business/wifi/imgs/wifi.jpg
    


    /*合并压缩后*/
    --asset/   
    ------config.js  //Seajs配置文件
    ------page   //页面级js
    ------page/share/share-{{md5}}.js
    ------page/share/share.css
    ------page/share/imgs/share.jpg
    ------page/share/imgs/wifi.jpg
    
share.js相对路径引用wifi.js，wifi.js相对路径引用wifi.css;构建工具处理后，share-{{md5}}.js,合并了wifi.js，share.css合并了wifi.css,imgs文件夹合并。

##什么是索引文件

在开发中，在jsp指定某个变量代表一个开发路径文件，

在生产环境中，jsp中相应的变量指向合并压缩后的路径文件
开发时索引关系文件 /square/src/config.json
发布时索引关系文件 /square/asset/config.json

##为什么用md5后缀命名压缩后文件

实现非覆盖式发布，如果使用 ?20141202 这种形式。上线时，不管是先更新静态服务器，还是先更新web服务器，用户在间隔空挡时间内访问页面，都会有问题。
另外使用md5命名文件后，可以将浏览器缓存时间设置成很长，非常有利于提升页面访问性能。
目前只对js文件实现了md5命名，下一步是实现图片和样式的文件名转换MD5
    
##widget目录

标准模块，放置如jquery,backbone,dialog之类的基础组件，采用版本控制，比如改了dialog.js，要同时手动改下package.json中的版本号。
Ps:这样设计时，是因为当时想使用spm包管理器。

处理前
  
        --src                          
        ------widget/dialog/
        ------widget/dialog/package.json
        ------widget/dialog/src/dialog.js
        ------widget/dialog/src/supportBase.js
        ------widget/dialog/src/dialog.css
        ------widget/dialog/src/imgs/tip.png
        ------widget/dialog/examples  --组件使用demo 

处理后
  
        --asset
        -----config.js --在配置文件中加入别名
           "dialog": "widget/dialog/0.0.1/dialog.js",
        -----widget/dialog/
        -----widget/dialog/0.0.1/dialog.js
        -----widget/dialog/0.0.1/dialog.css
        -----widget/dialog/0.0.1/imgs/tip.png


 
#使用说明

##下载安装

    npm install -g ysf@0.0.12 
    // 在命令行中，定位到square
    ysf build-widget     --处理widget目录下文件，asset/widget
    ysf build-page
    // 这样src中内容，全部打包到asset

       
##相关配置
（1）config.ini(Java配置文件) 
dev.front.swtich false 生产模式（测试环境、pb、线上的默认值）
true 开发模式（svn上默认值，本地服务、test.shipin.com）

shipin7.static.domain 生产模式下静态服务器的域名地址
assetsVersion ?20141125 静态资源的时间戳
（2）/square/cfgCenter.json(开发模式下前端配置文件)
compression false 不用压缩代码
true 用压缩代码
staticServer 静态资源从静态服务器上加载，为空时不使用静态服务器

##切换模式
测试、线上默认是采用合并压缩，切换到非压缩的步骤
（1）
[tomcat根目录]/appconfig/shipin7/config.ini
dev.front.swtich=false 改成 dev.front.swtich=true
（2）
[tomcat根目录]/webapps/ROOT/square/cfgCenter.json
改成如下值
{
"compression":false
// "staticServer":"static.shipin7.com"
}

##发布线上
视频广场采用静态服务器，要先更新静态服务器，再启动tomcat
如果运维先更新启动了tomcat，后更新静态服务器，提醒下运维刷下目前4台tomcat的缓存；或者打开浏览器 https://i.ys7.com/front/cacheView.action ，点击 更新缓存

##线上出现bug，如何调试？
在网址后面，加上请求参数debug=true
打开调试，看到的是压缩最后一步前的代码

##如何打补丁？
运行合并压缩命令，将有变动的压缩文件，发给运维，直接更新到静态服务器即可，注意：变动的文件除了合并压缩后文件，还有索引文件/square/asset/config.json

运维更新时，提醒下运维刷下目前4台tomcat的缓存；或者打开浏览器 https://i.ys7.com/front/cacheView.action ，点击 更新缓存

##有关缓存
图片、css线上设置的缓存时间是max-age=3600（1小时），如果更新文件没生效，可提醒运维改下时间戳config.ini  assetsVersion:?20141125
page下面的Js是根据文件内容md5命名，因此不用担心缓存问题
组件widget下，是由版本号防止缓存，如果修改了Js，要改下package.json的version

##开发者注意事项

1. 如何新增一个组件a

   （1）
   
       square/src/widget/a/src/a.js
                        /a/src/a.css
                        /a/src/imgs/
                        /a/package.json(组件相关信息)
                               
   如果a.js不是seajs模块格式，在文件最后新增 define(function(){});
   
   （2）square/src/config.json
   文件后加上
   
    a:"widget/a/src/a.js"
   
   (3) 其它Js调用时，直接
   
    require("a")
    
2.如何引用组件中图片

不推荐组件外的js/css引用组件下的图片

如果一定要用的话，比如其它Js这样使用

    seajs.resolve("jquery_emotionFace").match(/[\:\/\w\.]+\//) + 'imgs/$1.png"
    
3.引用组件级别模块，要使用别名

4.require非seajs格式的js，打包时不能解析

5.不要require了base下js

6.页面引入js时，需要seajs.use('page/index/index');
不能seajs.use('page/index/index.js')，是为了方便切换开发和压缩模式

7.不要增加文件层次结构
    page、business、widget的文件夹里只能放一级功能目录，
    
        --page
        ------/setting/
        --------------/product_s1
        -------------------------/product_s1.js
        -------------------------/product_s1.css
        -------------------------/imgs
        --------------/product_s2
        -------------------------/product_s2.js
        -------------------------/product_s2.css
        -------------------------/imgs
        
        应该改成这样
        
        --page
        ------/product_s1
        -----------------/product_s1.js
        -----------------/product_s1.css
        -----------------/imgs
        ------/product_s2
        -----------------/product_s2.js
        -----------------/product_s2.css
        -----------------/imgs

8.页面级css不能引用business下图片

            
#nodeJs构建工具的开发

##实现逻辑

###如何实现全局安装
package.json 配置 "bin":"bin/ysf"

    目录A: 命令行运行目录，比如D:/square下，运行ysf build-widget,目录A: D:/square
    目录B: 命令行工具所在路径，比如 C:\Users\administrator\AppData\Roaming\npm\node_modules\ysf
                  
ysf 基于grunt上开发的，ysf是安装在全局目录(目录B)，依赖的grunt和其它npm包也安装在全局目录(目录B);

ysf 调用grunt时，grunt默认处理的是目录B下的文件，需要切换到目录A;

为实现切换功能，修改grunt-->ysf_grunt

###如何实现grunt任务调用grunt
在对组件进行构建时，是以单个组件为单元，这个组件构建完成后，再进行下个组件构建。这里封装成`grunt-ingrunts`

    ysf build-widget时，调用grunt的beginWidget任务;
    beginWidget
    分析组件目录下所有组件，
    按配置排序后，
    取出排序后第一个组件，切换grunt上下文，再次调用grunt，加载执行Gruntfile.widget.js；
    构建组件生成目标文件后，将组件的文件路径和名称索引关系，写入config.json
    取出下一个组件...

###合并规则
spm的合并使用了`grunt-cmd-concat`。如果js里面require了css，默认会将css和js合并成一个文件，css引用的相对路径的图片资源会找不到。

在这里，按照js、css、imgs拆分成3个任务，

    合并js,采用`grunt-cmd-concatself`;
    合并css，采用`grunt-cmd-concatcss`，依据js的依赖声明，将依赖的css独立合并成一个css;
    合并imgs,采用`grunt-cmd-concatfile`，依据js的依赖声明，将有依赖关系的imgs图片合并成一个imgs。

##build-page构建流程
参考安装包中Gruntfile.js

      /* 1.清空：.build
       * 2.清空asset
       * 5.压缩base下指定的css,src/base/ --> asset/base
       * 6.拷贝base下所有图片,src/base/**.{jpg|png|jpeg|gif} --> asset/base/
       * 7.压缩base下所有的js,src/base/**.js --> asset/base/**.js
       * 8.拷贝base下剩下的文件,src/base/**.{otf,eot,svg,ttf,woff} --> asset/base/
       * 8.提取page目录下页面模块的id和依赖,src/page/**.js --> .build/page                                                                                                        
       * 9.提取business目录下页面模块的id和依赖,src/business/**.js --> .build/business
       * 10.拷贝page和business下css和图片，以协同作下一步处理,src/{business|page}/**.{png,jpg,jpeg,gif} --> .build/
       * 12.合并page依赖的business下的js文件,.build/page --> .build/concat
       * 11.合并page依赖的business下的css文件，到page下css文件中,.build/page --> .build/concat
       * 13.合并page依赖的business下的图片文件夹,.build/page --> .build/concat
       * 14.压缩合并的js,.build/concat/**.js --> .build/uglify/
       * 15.根据合并后js的md5值，重命令js文件,.build/uglify/**.js --> asset/page/
       * 16.压缩合并的css，.build/concat --> asset/page
       * 17.拷贝合并的imgs文件夹, .build/page/**.{jpg|png|jpeg|gif}a --> asset/page
       * 18.修改config.json配置文件*/

1. 压缩base下js、css、图片

2. transport
    
    使用`grunt-cmd-transport`，提取出id和依赖后，可放心合并、压缩。

		define(function(require,exports){}) 

		define(id,[dependences],function(require,exports,module){})

    要注意路径配置，容易出现使用合并压缩后的代码，资源加载了，却找不到执行路径。

3. 合并js
    
    `grunt-cmd-concat`，插件include属性值为relactive时，page目录下的Js，会合并require相对路径的js，别名的、绝对路径的不合并
    
	配置include参数值

	一种是self，不合并；一种relactive（默认），合并相对路径的依赖；一种all，合并所有依赖。
    

4. 合并css
    
    `grunt-cmd-concatcss`，合并的js涉及到的css都合并成一个css，放在页面级css中

5. 合并imgs
    
    `grunt-cmd-concatfile`，合并的js涉及到的imgs文件夹都合并成一个imgs，放在页面级imgs中

6. 压缩合并的Js
    
    `grunt-contrib-uglify`，

7. 压缩css
    
    `grunt-contrib-cssmin`

8. 压缩图片
    
    `grunt-contrib-imagemin`

9. 重命名合并压缩后的js
    
    根据MD5重命名，解决Js设置缓存后，版本更新不能及时生效

    在SeaJs配置文件，加入命名前后对照关系
    方便切换开发与线上环境模式

##build-widget流程
参考安装包中Gruntfile.widget.js

##开启调试
开启调试，将加载合并后未压缩的debug文件

在写合并压缩代码时，如何开启调试？

    先安装插件
    npm install -g node-inspector

    打开命令行，启动调试端口
    node-inspector

    打开另一个命令行窗口
    node --debug-brk $(which grunt) task
    node --debug-brk C:\Users\administrator\AppData\Roaming\npm\node_modules\grunt-cli\bin\grunt concat:app
    node --debug-brk C:\Users\administrator\AppData\Roaming\npm\node_modules\spm-build\bin\spm-build 

    最后打开chrome，输入网址 
    http://127.0.0.1:8080/debug?port=5858

打印具体细节

    ysf build-page --verbose


或者直接在代码里 grunt.log.writeln

#参考资料
[用Node.js创建命令行工具](http://www.html-js.com/article/2087)

[Seajs源码](https://github.com/seajs/seajs)

[Nodejs Api](http://nodejs.org/api/)

[Grunt文档及Api](http://gruntjs.com/)

[Spm构建与包管理](http://docs.spmjs.org/doc/)

