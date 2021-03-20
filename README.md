taro集成到现有原生应用

React Native组件集成到android应用有如下几个步骤：

1.配置好依赖和项目结构
2.创建taro web项目，编辑React Native组件的js代码
3.在应用中添加一个ReactRootView，这个ReactRootView是用来承载你的React Native组件的容器
4.启动taro web 的rn服务，运行应用
5.验证组件是否正常运行
6.离线打包jsbundle
1. 配置项目结构

下载Taro 原生 React Native 壳子，地址 https://github.com/NervJS/taro-native-shell
复制一份下载下来taro-native-shell项目命名为demo，删除目录下ios项目，复制需要集成的工程目录到taro-native-shell目录下，以Bestore为例子
把Bestore项目名称更改为android
启动命令行到demo目录下，执行npm install,待执行完成你会发现demo目录下多了一个文件夹node_modules，这个正是安卓项目依赖的库
androidstudio打开demo的android项目，当你发现This version of the Android Support plugin for IntelliJ IDEA (or Android Studio) cannot open this project, please retry with version 4.1 or newer这个错误的时候就要更改classpath 'com.android.tools.build:gradle:4.1.0-alpha09'了更改为classpath'com.android.tools.build:gradle:3.6.3'和gradle-wrapper.properties更改为distributionUrl=https://services.gradle.org/distributions/gradle-5.6.4-all.zip然后rebuildproject，保证app能正常运行
2.把 React Native 添加到你的应用中

配置 maven（参考官方壳子配置）

在你的 app 中 build.gradle 文件中添加 React Native 依赖:

  implementation "com.facebook.react:react-native:+"  // From node_modules
  // 基础依赖包，必须要依赖
  implementation 'com.gyf.immersionbar:immersionbar:3.0.0'
  // fragment快速实现（可选）
  implementation 'com.gyf.immersionbar:immersionbar-components:3.0.0'
  // kotlin扩展（可选）
  implementation 'com.gyf.immersionbar:immersionbar-ktx:3.0.0'
  // For animated GIF support
  implementation 'com.facebook.fresco:animated-gif:1.3.0'
和相关配置

apply from: "../../node_modules/react-native/react.gradle"
def enableSeparateBuildPerCPUArchitecture = false

 android {
 splits {
        abi {
            reset()
            enable enableSeparateBuildPerCPUArchitecture
            universalApk false  // If true, also generate a universal APK
            include "armeabi-v7a", "x86", "arm64-v8a", "x86_64"
        }
    }
    buildTypes {
        release {
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
        }
    }
    // applicationVariants are e.g. debug, release
    applicationVariants.all { variant ->
        variant.outputs.each { output ->
            // For each separate APK per architecture, set a unique version code as described here:
            // http://tools.android.com/tech-docs/new-build-system/user-guide/apk-splits
            def versionCodes = ["armeabi-v7a":1, "x86":2, "arm64-v8a": 3, "x86_64": 4]
            def abi = output.getFilter(OutputFile.ABI)
            if (abi != null) {  // null for the universal-debug, universal-release variants
                output.versionCodeOverride =
                        versionCodes.get(abi) * 1048576 + defaultConfig.versionCode
            }
        }
    }
}

// Run this once to be able to run the application with BUCK
// puts all compile dependencies into folder libs for BUCK to use
task copyDownloadableDepsToLibs(type: Copy) {
  from configurations.compile
  into 'libs'
}

在项目的 build.gradle 文件中为 React Native 添加一个 maven 依赖的入口，必须写在 "allprojects" 代码块中:

  maven {
        // All of React Native (JS, Obj-C sources, Android binaries) is installed from npm
        url("$rootDir/../node_modules/react-native/android")
      }
配置权限 如果需要访问 DevSettingsActivity 界面（即开发者菜单），则还需要在 AndroidManifest.xml 中声明:

<activity android:name="com.facebook.react.devsupport.DevSettingsActivity" />
下载需要的依赖，保证项目运行成功

setting.gradle添加如下配置

apply from: '../node_modules/react-native-unimodules/gradle.groovy'
include ':react-native-svg'
project(':react-native-svg').projectDir = new File(rootProject.projectDir, '../node_modules/react-native-svg/android')
includeUnimodulesProjects()

rootProject.name = 'taroDemo'
app下的build.gradle添加如下依赖

  implementation project(':react-native-svg')
   addUnimodulesDependencies()
你会发现又如下错误

FAILURE: Build completed with 2 failures.

1: Task failed with an exception.
-----------
* Where:
Script '/Users/jerry/Desktop/demo/node_modules/react-native-unimodules/gradle.groovy' line: 81

* What went wrong:
A problem occurred evaluating project ':app'.
> You need to have MainApplication in your project
我的做法是先注释掉 app gradle下的配置 addUnimodulesDependencies()和 apply from: '../../node_modules/react-native-unimodules/gradle.groovy'

然后根据官方demo 配置application 和入口activity

application


public class MainApplication extends Application implements ReactApplication {
  private final ReactModuleRegistryProvider mModuleRegistryProvider = new ReactModuleRegistryProvider(new BasePackageList().getPackageList(), Arrays.<SingletonModule>asList());

  private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
    @Override
    public boolean getUseDeveloperSupport() {
      return BuildConfig.DEBUG;
    }

    @Override
    protected List<ReactPackage> getPackages() {
      return Arrays.<ReactPackage>asList(
        new MainReactPackage(),
        new SvgPackage(),
        new ToastReactPackage(),
        new ModuleRegistryAdapter(mModuleRegistryProvider)
      );
    }

    @Override
    protected String getJSMainModuleName() {
      return "index";
    }
  };

  @Override
  public ReactNativeHost getReactNativeHost() {
    return mReactNativeHost;
  }

  @Override
  public void onCreate() {
    super.onCreate();
    SoLoader.init(this, /* native exopackage */ false);
  }
activity

public class MainActivity extends ReactActivity {

    /**
     * Returns the name of the main component registered from JavaScript.
     * This is used to schedule rendering of the component.
     */
    @Override
    protected String getMainComponentName() {
        return "RNBeStore";// 注意这里的RNBeStorep必须对应“index.js”中的
        // “AppRegistry.registerComponent()”的第一个参数
    }

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    <!--ImmersionBar.with(this)-->
    <!--  .statusBarColor(R.color.colorStatusBar)-->
    <!--  .statusBarDarkFont(true)-->
    <!--  .fitsSystemWindows(true)  //使用该属性必须指定状态栏的颜色，不然状态栏透明，很难看-->
    <!--  .init();-->
  }
}
public class BasePackageList {
  public List<Package> getPackageList() {
    return Arrays.<Package>asList(
        new expo.modules.av.AVPackage(),
        new expo.modules.brightness.BrightnessPackage(),
        new expo.modules.constants.ConstantsPackage(),
        new expo.modules.filesystem.FileSystemPackage(),
        new expo.modules.imagepicker.ImagePickerPackage(),
        new expo.modules.location.LocationPackage(),
        new expo.modules.permissions.PermissionsPackage(),
        new expo.modules.sensors.SensorsPackage()
    );
  }
}
//注意依赖的包名为import org.unimodules.core.interfaces.Package;
rebuild project 编译项目

根据爆红的代码添加相应的库依赖

编译代码报如下错误 Caused by: org.gradle.internal.component.NoMatchingConfigurationSelectionException: Unable to find a matching configuration of project :react-native-svg: 我顺着android的同级目录node_modules下找没有找到这个库

经过咨询得知taro工程统一引入了阿里的iconfont图标库引入方式：https://github.com/iconfont-cli/taro-iconfont-cli 在React-Native中，图标需要依赖react-native-svg，所以您需要在taro项目中安装这个插件

# Npm
npm install react-native-svg

# Yarn
yarn add react-native-svg
rebuild 果然项目编译成功

于是通过真机跑项目，出现了如下错误

/Users/jerry/Desktop/demo/node_modules/unimodules-image-loader-interface/android/src/main/java/org/unimodules/interfaces/imageloader/ImageLoader.java:5: 错误: 程序包android.support.annotation不存在
import android.support.annotation.Nullable;

由于原生项目support库已经全部升级为Androidx，所以所有引入了android support库的 依赖都要改为androidx 依赖

待解决完冲突后出现如下错误：

uses-sdk:minSdkVersion 17 cannot be smaller than version 21 declared in library [:expo-av] /Users/jerry/Desktop/demo/node_modules/expo-av/android/build/intermediates/library_manifest/debug/AndroidManifest.xml as the library might be using APIs not available in 17

按照提示是更改sdk最低版本为21
更改后重新编译报

/Users/jerry/Desktop/demo/android/app/src/main/java/com/lppz/mobile/android/outsale/pubfunction/pubutils/ImagPagerUtil.java:28: 错误: 程序包com.bumptech.glide.request.animation不存在
import com.bumptech.glide.request.animation.GlideAnimation;
通过分析得知，依赖的库和app项目引入图片的依赖不一致导致 二者必须统一图片加载库glide的版本号

由于app项目升级glide版本代价太大，于是只有降低依赖库的版本

再次编译

/Users/jerry/Desktop/demo/node_modules/@unimodules/react-native-adapter/android/src/main/java/org/unimodules/adapters/react/services/UIManagerModuleWrapper.java:17: 错误: 程序包com.bumptech.glide.request.transition不存在
import com.bumptech.glide.request.transition.Transition;
解决冲突代码

Glide.with(getContext())
        .asBitmap()
        .diskCacheStrategy(DiskCacheStrategy.NONE)
        .skipMemoryCache(true)
        .load(url)
        .into(new SimpleTarget<Bitmap>() {
          @Override
          public void onResourceReady(@NonNull Bitmap resource, @Nullable Transition<? super Bitmap> transition) {
            resultListener.onSuccess(resource);
          }

          @Override
          public void onLoadFailed(@Nullable Drawable errorDrawable) {
            resultListener.onFailure(new Exception("Loading bitmap failed"));
          }
        });
更改为

  Glide.with(getContext())
      .load(url)
      .asBitmap()
      .diskCacheStrategy(DiskCacheStrategy.NONE)
      .skipMemoryCache(true)
      .into(new SimpleTarget<Bitmap>() {
        @Override
        public void onResourceReady(Bitmap resource, GlideAnimation<? super Bitmap> glideAnimation) {
          resultListener.onSuccess(resource);
        }

//        @Override
//        public void onResourceReady(@NonNull Bitmap resource, @Nullable Transition<? super Bitmap> transition) {
//          resultListener.onSuccess(resource);
//        }

        @Override
        public void onLoadFailed(Exception e, Drawable errorDrawable) {
//          super.onLoadFailed(e, errorDrawable);
          resultListener.onFailure(new Exception("Loading bitmap failed"));
        }

//        @Override
//        public void onLoadFailed(@Nullable Drawable errorDrawable) {
//          resultListener.onFailure(new Exception("Loading bitmap failed"));
//        }
      });
rebuild并运行到真机上

大公告成

3 创建taro web项目，编写js

此部分省略（参考凯哥web项目） 直接运行项目

npm install -g @tarojs/cli
yarn
yarn dev:rn
4.验证组件是否正常运行

打开app找到承载react的activity
摇一摇手机
配置服务器地址
reload
5 在应用中添加一个ReactRootView，这个ReactRootView是用来承载你的React Native组件的容器

由于全局配置react ，导致多次打开react 页面导致app闪退 于是做了如下调整

public class MyActivity extends ReactBaseActivity implements DefaultHardwareBackBtnHandler {

  /**
   * Returns the name of the main component registered from JavaScript. This is used to schedule
   * rendering of the component.
   */
//    @Override
//    protected String getMainComponentName() {
//        return "taroDemo";
//    }

  private ReactRootView mReactRootView;
  private ReactInstanceManager mReactInstanceManager;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    mReactRootView = new ReactRootView(this);
    mReactInstanceManager = ReactInstanceManager.builder()
      .setApplication(getApplication())
      .setCurrentActivity(this)
      .setBundleAssetName("index.android.bundle")
//      .setJSMainModulePath("index")
      .setJSMainModulePath("rn_temp/index")
      .addPackages(Arrays.<ReactPackage>asList(
        new MainReactPackage(),
        new SvgPackage(),
        new ToastReactPackage(),
        new ModuleRegistryAdapter(mModuleRegistryProvider)
      ))
      .setUseDeveloperSupport(BuildConfig.DEBUG)
      .setInitialLifecycleState(LifecycleState.RESUMED)
      .build();
    // 注意这里的MyReactNativeApp必须对应“index.js”中的
    // “AppRegistry.registerComponent()”的第一个参数
//    mReactRootView.startReactApplication(mReactInstanceManager, "RNBeStore", null);

    mReactRootView.startReactApplication(mReactInstanceManager, "taroDemo", null);


    setContentView(mReactRootView);
  }
 private final ReactModuleRegistryProvider mModuleRegistryProvider = new ReactModuleRegistryProvider(new BasePackageList().getPackageList(), Arrays.<SingletonModule>asList());

  private final ReactNativeHost mReactNativeHost = new ReactNativeHost(getApplication()) {
    @Override
    public boolean getUseDeveloperSupport() {
      return BuildConfig.DEBUG;
    }

    @Override
    protected List<ReactPackage> getPackages() {
      return Arrays.<ReactPackage>asList(
        new MainReactPackage(),
        new SvgPackage(),
        new ModuleRegistryAdapter(mModuleRegistryProvider)
      );

    }
    @Override
    protected String getJSMainModuleName() {
      return "rn_temp/index";
    }
  };
  @Override
  public void invokeDefaultOnBackPressed() {
    super.onBackPressed();
  }

  @Override
  protected void onPause() {
    super.onPause();

    if (mReactInstanceManager != null) {
      mReactInstanceManager.onHostPause(this);
    }
  }

  @Override
  protected void onResume() {
    super.onResume();

    if (mReactInstanceManager != null) {
      mReactInstanceManager.onHostResume(this, this);
    }
  }

  @Override
  protected void onDestroy() {
    super.onDestroy();

    if (mReactInstanceManager != null) {
      mReactInstanceManager.onHostDestroy(this);
    }
    if (mReactRootView != null) {
      mReactRootView.unmountReactApplication();
    }
  }

  @Override
  public boolean onKeyUp(int keyCode, KeyEvent event) {
    if (keyCode == KeyEvent.KEYCODE_MENU && mReactInstanceManager != null) {
      mReactInstanceManager.showDevOptionsDialog();
      return true;
    }
    return super.onKeyUp(keyCode, event);
  }
}

去掉application的配置仅仅保留初始化 SoLoader.init(this, /* native exopackage */ false);

6.离线打包jsbundle

你也可以使用 Android Studio 来打 release 包！其步骤基本和原生应用一样，只是在每次编译打包之前需要先执行 js 文件的打包(即生成离线的 jsbundle 文件)。具体的 js 打包命令如下：

$ react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/com/your-company-name/app-package-name/src/main/assets/index.android.bundle --assets-dest android/com/your-company-name/app-package-name/src/main/res/

注意把上述命令中的路径替换为你实际项目的路径。如果 assets 目录不存在，则需要提前自己创建一个。

以我们的taro web项目为例子

用webstorm打开web项目
web目录下新建bundle文件夹
执行以下命令
react-native bundle --entry-file rn_temp/index.js --platform android --dev false --bundle-output ./bundle/index.android.bundle --assets-dest ./bundle/
生成index.android.bundle文件和图片资源文件

把index.android.bundle放到app的assets资源文件中，把图片资源放到app下的res文件夹中
运行安卓项目，大公告成
