# 组件化技术文档

## 总体框架


   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E7%BB%84%E4%BB%B6%E5%8C%96%E6%A1%86%E6%9E%B6.png"
width="600"
/>



## 基础服务层
基础服务层是项目的基础架构，目前包含了4个模块，从下到上形成一个模块依赖链。

*作用：*

该层是项目的底层架构，为所有的业务模块提供公共的服务。

*规范*

该层只提供通用的服务和组件。


#### lib_tools

底层工具：提供通用的**与android无关的java方法**。如：日志，加密。目标是聚合通用方法，能够以jar格式，在公司各个app项目之间通用。
	
>    Deprecated:由于项目的迭代和技术的发展，现阶段很难规定一些不会改变的底层方法，能够一直的服务于所有的项目。比如现有的LogUtil方法，很可能在未来改造其实现，添加其他功能，比如：异步输出，本地缓存...该模块中的内容将会移动到lib_basic_module中，作为对外的服务。

####  lib_domain

底层模型：提供最底层的数据模型。通用的数据模型通常会在各个业务模块间通用的数据模型会放在该层中。**子业务模块独有的数据模型不能放在该层。**可以放在其中的数据模型如下图：
	

#### lib_domain包括：


   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/lib_domain_java.png"
/>


domain:业务数据模型。
event:rxBux事件
request：网络请求数据 
response：网络返回数据
	
####  lib_rpc
   
   模块间调用连接器。架构如下：
   
   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/rpc%E6%A1%86%E6%9E%B6.png"
width="800"
/>
   
   简单来说，要实现从其他模块调用ModuleA，需要如下几步：

	1. 在rpc的"com.maihaoche.bentley.rpc.module"包下创建一个接口IAModule，继承IModule。该接口的方法为目标模块A对外开放的API；
	
	2.让模块A依赖lib_rpc_module，并且在模块A中创建IAModule的实现类ImpAModule;
	
	3.在lib_rpc_module的ModuleServiceManager中定义类ImpAModule的全路径常量和调用方法。例：
   
		private final static String MODULE_A = "com.maihaoche.bentley.ImpAModule";
		
		public static IAModule getAModule() throws ModuleNotAssembledException {
        	return getModule(MODULE_A);
    	}
           	
	4.在其他模块中，依赖lib_rpc_module,然后就可以调用getAModule的方法，来调用ModuleA中的API了。
	
####  lib_basic_module

基础层：包含了项目的基础服务+模块的基础架构。**提供全局的与具体业务无关的组件方法**.

基础服务包括：数据库，图片加载，网络调用，tinker补丁，定位，埋点，toast提示，推送，权限...等等.
基础架构包括:Activity Fragment的基类，adapter的基类，各种控件的基类，appliatoin的基类...等



## 公共业务层

*作用：*

将项目的通用业务进行整合，解构，供上层依赖调用。

*规范*

该层只包含业务相关的服务，并且是各个模块间**公用**。**独立业务模块单独的业务不要放在该层**


#### lib_basic_biz
与lib_basic_module的功能类似，**区别是：提供全局的业务息息相关的服务**。
例如：分享，权限认证，选择车型，选择城市，推送跳转等等。这些服务的特点是：各个子业务模块都频繁调用，并且这些模块代码量较小。

#### lib_photo
该模块(图片浏览和图片选择)实际上应该归属于lib_basic_biz中，但是由于其代码较多，单独分出来形成一个独立的业务子模块，供上层业务调用，实现代码分离和条理清晰。

#### lib_update
提供app升级的业务子模块。与lib_photo定位相同。


## 业务模块层
##### module_auth：认证业务
##### module_dms：dms业务
##### module_loan：订单贷业务
##### module_logistics：物流业务
##### module_msg：消息业务
##### module_pay：支付业务
##### module_seek：寻车业务
##### module_source_new：车源业务
##### module_trade：交易业务

## Build.gradle统一配置
分模块后，build.gradle的文件也需要统一管理。具体实践：
1.所有子模块的build.gradle 文件都必须在开头引用lib_gradle_scripts模块中的subModuleHeader.gradle文件；
2.由于subModuleHeader.gradle完成了所有的通用配置，包括版本，签名，混淆等和dependency中的部分通用依赖。所以引用了之后，子模块的build.gradle只需要填写自身特有的配置，比如manifestPlaceholders，以及独有的依赖。
以寻车的build.gradle为例：
		
	//依赖subModuleHeader.gradle文件
	apply from: "$rootProject.projectDir/$LIB_GRADLE_SCRIPTS/subModuleHeader.gradle"

	android {
	    resourcePrefix 'seek_'
	    buildTypes {
	        release {
	            manifestPlaceholders = [
	                    schemeName: "activity"
	            ]
	        }
	        debug {
	            manifestPlaceholders = [
	                    schemeName: "activityd"
	            ]
	        }
	    }
	}
	//要是使用commonDependency方法，需要把	DependencyHandler作为参数传入。
	dependencies { org.gradle.api.artifacts.dsl.DependencyHandler dependencyHandler ->
	    commonDependency(dependencyHandler, [LIB_BASIC_BIZ,LIB_BASIC_MODULE,LIB_DOMAIN,LIB_RPC,LIB_TOOLS,LIB_PHOTO] as String[])
	}
	//single app的单独配置项。当该子模块配置了独立运行的逻辑时，需要改方法使其生效。
	globleConfigure()

## 模块独立运行
项目模块数增长后，开发时的编译就会非常耗时。为了提高开发时的效率，设计了一种以AAR依赖为核心的设计，使得我们在开发的过程中，可以让某个业务模块作为独立的app运行，大大提高开发效率。
从多模块编译，到独立模块编译的示意图如下：

   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E7%8B%AC%E7%AB%8B%E6%A8%A1%E5%9D%97%E7%A4%BA%E6%84%8F%E5%9B%BE.png"
width="700"
/>

除了手动进行该配置外，我们优化了独立模块运行的切换手段：

* 使用gradle 的task，在控制台运行带参数的脚本任务。
* 使用插件配合gradle任务，进行可视化操作。

> 由于脚本都是gradle的一些语法，所以不做具体介绍了。可以参考工程根目录下的build.gradle中定义的turnToSingleModule 这个task及其注解。下面就简单介绍一下我们的插件。


## 单Module插件助手

[MazDa](https://github.com/WangYangYang2014/MaiHaoCheMazDa)

该插件是与脚本配合使用，实现项目中全Module<——>单Module之间快速自动化切换，并配合图形化交互页面，实现可配置和本地缓存。

使用教程：

1. 从Android Studio内置插件浏览器中搜索Mazda,并安装重启：

   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E6%90%9C%E7%B4%A2mazda.png"
width="800"
/>

2. 安装完插件并且重新启动Android Studio后，再次启动进来，并等项目加载完毕后，点击菜单Tools，下拉列表中会多出一个Mazda的选项，点击并在展开的列表中选择Configure：

   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E9%80%89%E6%8B%A9%E9%85%8D%E7%BD%AE.png"
width="600"
/>


3. 配置参数。点击Configure后会弹出如下的参数设置窗口：

  <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E9%85%8D%E7%BD%AE.png"
width="400"
/>
 

4. 切换到全Module运行状态：点击Tools->Mazda->To All。等待任务运行完毕即可。

5. 切换到单Module运行状态。需要如下几个步骤

	*  点击Tools——>Mazda——>To Single。稍等一会后，会弹出如下的对话框，该单选对话框用于选择需要独立运行的模块：
	
   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E5%8D%95module%E9%80%89%E6%8B%A9.png"
width="200"
/>
	*  接来下会弹出如下对话框，该复选对话框是选择需要以AAR形式添加到之前的单module的依赖中：
	
   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E9%80%89%E6%8B%A9%E4%BE%9D%E8%B5%96.png"
width="200"
/>
	*  然后等待任务执行完成，最后Android Studio会弹出如下提示窗口，点击确定即可：
	

   <img 
src= "http://of8cu1h2w.bkt.clouddn.com/%E7%A1%AE%E5%AE%9A.png"
width="400"
/>



## 存在的问题和新的方向
#### 问题
现阶段的组件化方案，既满足了代码分离解构，同时又加快了开发效率，能够较好得支持团队业务的推进。但是仍然存在几个问题:

1、rpc的核心是使用反射的手段，使不同的模块间能够进行方法调用。方法调用有一个缺陷是，对于异步的请求，不能够实现很好的回调。

>   现在一个中庸的解决方案是：在定义模块的对外接口的时候，方法中传入一个Action，作为调用获得结果后的回调处理。而回调则是通过RxBus来定义event来实现的。且该RxBus得subscrption需要绑定到外部调用真者的activity的生命周期中(确保该activity被销毁后，不会执行回调)。例子：ISeekModule中的createDepositModifyIntentOnPublishSource方法).
    该方案依然会带来问题：1.较多的event定义类；2.代码碎片化(回调处要post一个event出去)；3.调用者必须传入activity用于subscription的生命周期管理。

2、lib_basic_biz中聚集了很多的小的通用业务类，还有方法类等。在以后业务逐庞大后，该module难以满足大家的需求。 现阶段由于业务体量还不是很大，所以该设计轻巧便捷。若日后通用的业务组件加多的时候，我们可以考虑把这块分一分，既把业务分组。
 
#### 方向
Android的终极解决方案就要来了，那就是Kotlin！

有了Kotlin，rpc的设计更加优雅，参数无需对象类型，可通过类型推导。
有了Kotlin，代码更加简洁优雅，甚至期望出现流式异步编程，回调将不再需要。
有了Kotlin，开发效率更高，代码可读性更好。
所以我们的口号是：学好Kotlin，一个顶两没问题！


## 总结

组件化告一段落，全面向Kotlin过渡。






