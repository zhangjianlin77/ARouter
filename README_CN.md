```
    Android平台中对页面、服务提供路由功能的中间件
```

#### Look at here

##### [Demo apk](http://public.cdn.zhilong.me/app-debug.apk)
##### [![Join the chat at https://gitter.im/alibaba/ARouter](https://badges.gitter.im/alibaba/ARouter.svg)](https://gitter.im/alibaba/ARouter?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

#### 最新版本

###### arouter-annotation : [![Download](https://api.bintray.com/packages/zhi1ong/maven/arouter-annotation/images/download.svg)](https://bintray.com/zhi1ong/maven/arouter-annotation/_latestVersion)
###### arouter-compiler : [![Download](https://api.bintray.com/packages/zhi1ong/maven/arouter-compiler/images/download.svg)](https://bintray.com/zhi1ong/maven/arouter-compiler/_latestVersion)
###### arouter-api : [![Download](https://api.bintray.com/packages/zhi1ong/maven/arouter-api/images/download.svg)](https://bintray.com/zhi1ong/maven/arouter-api/_latestVersion)

#### Gradle依赖
```
dependencies {
    apt 'com.alibaba:arouter-compiler:x.x.x'
    compile 'com.alibaba:arouter-api:x.x.x'
}
```

![Demo gif](https://raw.githubusercontent.com/alibaba/ARouter/master/demo/arouter-demo.gif)

#### 一、功能介绍
1. **支持直接解析URL进行跳转、参数按类型解析，支持Java基本类型**
2. 支持应用内的标准页面跳转，API接近Android原生接口
3. **支持多模块工程中使用**
4. **支持跳转过程中插入自定义拦截逻辑，自定义拦截顺序**
5. **支持服务托管，通过ByName,ByType两种方式获取服务实例**
6. 映射关系按组分类、多级管理，按需初始化，减少内存占用提高查询效率(*)
7. 支持用户指定全局降级与局部降级策略
8. 被ARouter管理的页面、拦截器、服务均无需主动注册到ARouter

#### 二、不支持的功能
1. 自定义URL解析规则(考虑支持)
2. 不能动态加载代码模块和添加路由规则(考虑支持)
3. 多路径支持(不想支持，貌似是导致各种混乱的起因)
4. 生成映射关系文档(正在支持)

#### 三、典型应用场景
1. 从外部URL映射到内部页面，以及参数传递与解析
2. 跨模块页面跳转，模块间解耦
3. 拦截跳转过程，处理登陆、埋点等逻辑
4. 跨模块API调用，模块间解耦(注册ARouter服务的形式，通过接口互相调用)

#### 四、基础功能
1. 添加依赖和配置

        // 如gradle plugin >= 2.2，可以使用annotationProcessor，配置方法见文末'其他#4'
		apply plugin: 'com.neenbedankt.android-apt'

        buildscript {
            repositories {
                jcenter()
            }

            dependencies {
                classpath 'com.neenbedankt.gradle.plugins:android-apt:1.4'
            }
        }

        apt {
            arguments {
                moduleName project.getName();
            }
        }

        dependencies {
            compile 'com.alibaba:arouter-api:x.x.x'
            apt 'com.alibaba:arouter-compiler:x.x.x'
            ...
        }

2. 添加注解

		// 在支持路由的页面、服务上添加注解(必选)
		// 这是最小化配置，后面有详细配置, 这里的路径需要注意的是至少需要有两级，/xx/xx
        // 第一段用于分组，后面会详细提到
		@Route(path = "/test/1")
		public class YourActivity extend Activity {
		    ...
		}

3. 初始化SDK

        ARouter.init(mApplication); // 尽可能早，推荐在Application中初始化

4. 发起路由操作

		// 1. 应用内简单的跳转(通过URL跳转在'中阶使用'中)
		ARouter.getInstance().build("/test/1").navigation();

		// 2. 跳转并携带参数
		ARouter.getInstance().build("/test/1")
					.withLong("key1", 666L)
					.withString("key3", "888")
					.navigation();

5. 添加混淆规则(如果使用了Proguard)

        -keep public class com.alibaba.android.arouter.routes.**{*;}

#### 五、进阶用法
1. 通过URL跳转

        // 新建一个Activity用于监听Schame事件
        // 监听到Schame事件之后直接传递给ARouter即可
        // http://www.example.com/test/1
        public class SchameFilterActivity extends Activity {
            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);

                Uri uri = getIntent().getData();
                ARouter.getInstance().build(uri).navigation();
                finish();
            }
        }

        // AndroidManifest.xml
        <activity android:name=".activity.SchameFilterActivity">
                <!-- Schame -->
                <intent-filter>
                    <data
                        android:host="m.aliyun.com"
                        android:scheme="arouter"/>

                    <action android:name="android.intent.action.VIEW"/>

                    <category android:name="android.intent.category.DEFAULT"/>
                    <category android:name="android.intent.category.BROWSABLE"/>
                </intent-filter>

                <!-- App Links -->
                <intent-filter android:autoVerify="true">
                    <action android:name="android.intent.action.VIEW"/>

                    <category android:name="android.intent.category.DEFAULT"/>
                    <category android:name="android.intent.category.BROWSABLE"/>

                    <data
                        android:host="m.aliyun.com"
                        android:scheme="http"/>
                    <data
                        android:host="m.aliyun.com"
                        android:scheme="https"/>
                </intent-filter>
        </activity>

2. 使用ARouter协助解析参数类型

        // URL中的参数会默认以String的形式保存在Bundle中
        // 如果希望ARouter协助解析参数(按照不同类型保存进Bundle中)
        // 只需要在需要解析的参数上添加 @Param 注解
        @Route(path = "/test/1")
        public class Test1Activity extends Activity {
            @Param                   // 声明之后，ARouter会从URL中解析对应名字的参数，并按照类型存入Bundle
            public String name;
            @Param
            private int age;
            @Param(name = "girl")   // 可以通过name来映射URL中的不同参数
            private boolean boy;

            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);

                name = getIntent().getStringExtra("name");
                age = getIntent().getIntExtra("age", -1);
                boy = getIntent().getBooleanExtra("girl", false);   // 注意：使用映射之后，要从Girl中获取，而不是boy
            }
        }

3. 开启ARouter参数自动注入(实验性功能，不建议使用，正在开发保护策略)

        // 首先在Application中重写 attachBaseContext方法，并加入ARouter.attachBaseContext();
        @Override
        protected void attachBaseContext(Context base) {
           super.attachBaseContext(base);

           ARouter.attachBaseContext();
        }

        // 设置ARouter的时候，开启自动注入
        ARouter.enableAutoInject();

        // 至此，Activity中的属性，将会由ARouter自动注入，无需 getIntent().getStringExtra("xxx")等等

4. 声明拦截器(拦截跳转过程，面向切面搞事情)

        // 比较经典的应用就是在跳转过程中处理登陆事件，这样就不需要在目标页重复做登陆检查

        // 拦截器会在跳转之间执行，多个拦截器会按优先级顺序依次执行
        @Interceptor(priority = 666, name = "测试用拦截器")
        public class TestInterceptor implements IInterceptor {
            /**
             * The operation of this interceptor.
             *
             * @param postcard meta
             * @param callback cb
             */
            @Override
            public void process(Postcard postcard, InterceptorCallback callback) {
                ...

                callback.onContinue(postcard);  // 处理完成，交还控制权
                // callback.onInterrupt(new RuntimeException("我觉得有点异常"));      // 觉得有问题，中断路由流程

                // 以上两种至少需要调用其中一种，否则会超时跳过
            }

            /**
             * Do your init work in this method, it well be call when processor has been load.
             *
             * @param context ctx
             */
            @Override
            public void init(Context context) {

            }
        }

5. 处理跳转结果

		// 通过两个参数的navigation方法，可以获取单次跳转的结果
		ARouter.getInstance().build("/test/1").navigation(this, new NavigationCallback() {
            @Override
            public void onFound(Postcard postcard) {
			      ...
            }

            @Override
            public void onLost(Postcard postcard) {
                ...
            }
        });

6. 自定义全局降级策略

			// 实现DegradeService接口，并加上一个Path内容任意的注解即可
	       @Route(path = "/xxx/xxx") // 必须标明注解
			public class DegradeServiceImpl implements DegradeService {
			  /**
			   * Router has lost.
			   *
			   * @param postcard meta
			   */
			  @Override
			  public void onLost(Context context, Postcard postcard) {
			        // do something.
			  }

			  /**
			   * Do your init work in this method, it well be call when processor has been load.
			   *
			   * @param context ctx
			   */
			  @Override
			  public void init(Context context) {

			  }
			}

7. 为目标页面声明更多信息

		// 我们经常需要在目标页面中配置一些属性，比方说"是否需要登陆"之类的
		// 可以通过 Route 注解中的 extras 属性进行扩展，这个属性是一个 int值，换句话说，单个int有4字节，也就是32位，可以配置32个开关
		// 剩下的可以自行发挥，通过字节操作可以标识32个开关，通过开关标记目标页面的一些属性，在拦截器中可以拿到这个标记进行业务逻辑判断
		@Route(path = "/test/1", extras = Consts.XXXX)

8. 使用ARouter管理服务(一) 暴露服务

        /**
         * 声明接口
         */
        public interface IService extends IProvider {
            String hello(String name);
        }

        /**
         * 实现接口
         */
        @Route(path = "/service/1", name = "测试服务")
        public class ServiceImpl implements IService {

            @Override
            public String hello(String name) {
                return "hello, " + name;
            }

            /**
             * Do your init work in this method, it well be call when processor has been load.
             *
             * @param context ctx
             */
            @Override
            public void init(Context context) {

            }
        }

9. 使用ARouter管理服务(二) 发现服务

        1. 可以通过两种API来获取Service，分别是ByName、ByType
        IService service = ARouter.getInstance().navigation(IService.class);    //  ByType
        IService service = (IService) ARouter.getInstance().build("/service/1").navigation(); //  ByName

        service.hello("zz");

        2. 注意：推荐使用ByName方式获取Service，ByType这种方式写起来比较方便，但如果存在多实现的情况时，SDK不保证能获取到你想要的实现

10. 使用ARouter管理服务(三) 管理依赖

            可以通过ARouter service包装您的业务逻辑或者sdk，在service的init方法中初始化您的sdk，不同的sdk使用ARouter的service进行调用，
        每一个service在第一次使用的时候会被初始化，即调用init方法。
            这样就可以告别各种乱七八糟的依赖关系的梳理，只要能调用到这个service，那么这个service中所包含的sdk等就已经被初始化过了，完全不需要
        关心各个sdk的初始化顺序。

#### 六、更多功能

1. 初始化中的其他设置

		ARouter.openLog();	// 开启日志
		ARouter.printStackTrace(); // 打印日志的时候打印线程堆栈

2. 详细的API说明

		// 构建标准的路由请求
		ARouter.getInstance().build("/home/main").navigation();

		// 构建标准的路由请求，并指定分组
		ARouter.getInstance().build("/home/main", "ap").navigation();

		// 构建标准的路由请求，通过Uri直接解析
		Uri uri;
		ARouter.getInstance().build(uri).navigation();

		// 构建标准的路由请求，startActivityForResult
		// navigation的第一个参数必须是Activity，第二个参数则是RequestCode
		ARouter.getInstance().build("/home/main", "ap").navigation(this, 5);

		// 直接传递Bundle
		Bundle params = new Bundle();
		ARouter.getInstance()
					.build("/home/main")
					.with(params)
					.navigation();

		// 指定Flag
		ARouter.getInstance()
					.build("/home/main")
					.withFlags();
					.navigation();

		// 觉得接口不够多，可以直接拿出Bundle赋值
		ARouter.getInstance()
		            .build("/home/main")
		            .getExtra();

	    // 使用绿色通道(跳过所有的拦截器)
	    ARouter.getInstance().build("/home/main").greenChannal().navigation();

	    // 使用自己的日志工具打印日志
	    ARouter.setLogger();

3. 获取原始的URI

        String uriStr = getIntent().getStringExtra(ARouter.RAW_URI);

4. 重写跳转URL

       	// 实现PathReplaceService接口，并加上一个Path内容任意的注解即可，ARouter会直接处理经过用户处理之后的路径
        @Route(path = "/xxx/xxx") // 必须标明注解
        public class PathReplaceServiceImpl implements DegradeService {
            /**
             * For normal path.
             *
             * @param path raw path
             */
            String forString(String path) {
                return path;    // 按照一定的规则处理之后返回处理后的结果
            }

            /**
             * For uri type.
             *
             * @param uri raw uri
            */
            Uri forUri(Uri uri) {
                return url;    // 按照一定的规则处理之后返回处理后的结果
            }
         }

#### 七、其他

1. 路由中的分组概念

	- SDK中针对所有的路径(/test/1 /test/2)进行分组，分组只有在分组中的某一个路径第一次被访问的时候，该分组才会被初始化
	- 可以通过 @Route 注解主动指定分组，否则使用路径中第一段字符串(/*/)作为分组
	- 注意：一旦主动指定分组之后，应用内路由需要使用 ARouter.getInstance().build(path, group) 进行跳转，手动指定分组，否则无法找到

	        @Route(path = "/test/1", group = "app")

2. 拦截器和服务的异同

	- 拦截器和服务所需要实现的接口不同，但是结构类似，都存在 init(Context context) 方法，但是两者的调用时机不同
	- 拦截器因为其特殊性，会被任何一次路由所触发，拦截器会在ARouter初始化的时候异步初始化，如果第一次路由的时候拦截器还没有初始化结束，路由会等待，直到初始化完成。
	- 服务没有该限制，某一服务可能在App整个生命周期中都不会用到，所以服务只有被调用的时候才会触发初始化操作

3. Jack 编译链的支持

	- ~~因为不想让用户主动设置一堆乱七八糟的参数，在获取模块名的时候使用javac的api，使用了Jack之后没有了javac，只能让用户稍稍动动手了~~
	- 因为一些其他原因，现在任何情况下都需要在build.gradle中配置moduleName了。。。。

4. annotationProcessor的配置方式

        // 如果使用gradle plugin >= 2.2 可以不依赖android-apt，按照如下配置方法
        android {
            compileSdkVersion Integer.parseInt(COMPILE_SDK_VERSION)
            buildToolsVersion BUILDTOOLS_VERSION

            defaultConfig {
                ...
                javaCompileOptions {
                    annotationProcessorOptions {
                        arguments = [ moduleName : project.getName() ]
                    }
                }
            }
        }

        dependencies {
            compile 'com.alibaba:arouter-api:x.x.x'
            annotationProcessor 'com.alibaba:arouter-compiler:x.x.x'
            ...
        }

#### 八、Q&A

1. "W/ARouter::: ARouter::No postcard![ ]"

    这个Log正常的情况下也会打印出来，如果您的代码中没有实现DegradeService和PathReplaceService的话，因为ARouter本身的一些功能也依赖
    自己提供的Service管理功能，ARouter在跳转的时候会尝试寻找用户实现的PathReplaceService，用于对路径进行重写(可选功能)，所以如果您没有
    实现这个服务的话，也会抛出这个日志

    推荐在app中实现DegradeService、PathReplaceService

2. "W/ARouter::: ARouter::There is no route match the path [/xxx/xxx], in group [xxx][ ]"

    - 通常来说这种情况是没有找到目标页面，目标不存在
    - 如果这个页面是存在的，那么您可以按照下面的步骤进行排查
        1. 检查目标页面的注解是否配置正确，正确的注解形式应该是 (@Route(path="/test/test"), 如没有特殊需求，请勿指定group字段，废弃功能)
        2. 检查目标页面所在的模块的gradle脚本中是否依赖了 arouter-compiler sdk (需要注意的是，要使用apt依赖，而不是compile关键字依赖)
        3. 检查编译打包日志，是否出现了形如 ARouter::Compiler >>> xxxxx 的日志，日志中会打印出发现的路由目标
        4. 启动App的时候，开启debug、log(openDebug/openLog), 查看映射表是否已经被扫描出来，形如 D/ARouter::: LogisticsCenter has already been loaded, GroupIndex[4]，GroupIndex > 0

3. Instant Run

    - 请注意，ARouter和Instant Run并没有任何关系，也不会互相干扰，但是在开启Instant Run的情况下，编译期生成的映射文件，可能不会被Instant Run加载到apk中，
      而且在App没有重新初始化ARouter的情况下，ARouter并不会动态加载新生成的映射文件，如果新增注解，请全量编译并重新安装，鉴于开发中不会频繁添加注解，短期内不会
      对这种场景进行优化。

4. 沟通和交流

    ![qq](https://raw.githubusercontent.com/alibaba/ARouter/master/demo/arouter-qq-addr.png)