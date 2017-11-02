# Context #

Context经常称为"上下文"

关系如图：

![](https://i.imgur.com/zM058dL.png)

可以看出Context只是个抽象类，而实现类则是ContextIml

这里主要是使用了装饰者模式


## 基本介绍 ##

### ContextIml ###
Context的实现类，Context的相关方法的具体实现就是该类提供


### ContextWrapper ##

只是对Context的一种包装，里面含有了个mBase

	public class ContextWrapper extends Context {
    Context mBase; //该属性指向一个ContextIml实例，一般在创建Application、Service、Activity时赋值 

	 //创建Application、Service、Activity，会调用该方法给mBase属性赋值  
    public ContextWrapper(Context base) {
        mBase = base;
    }
    ...

### ContextThemeWrapper ##
ContextThemeWrapper继承ContextWrapper
主要是包含了主题(Theme)相关的接口

可以看出Activity是继承这个类需要主题，而Service是没有继承的


## Context的创建时期 ##

 情况有如下几种情况：

 1. 创建Application 对象时， 而且整个App共一个Application对象
 2. 创建Service对象时
 3. 创建Activity对象时
 
**总Context实例个数 = Service个数 + Activity个数 + 1（Application对应的Context实例）**

### 创建Application关联Context ###

创建Application的时候可以在ActivityThread中

	private void handleBindApplication(AppBindData data) {
	...
	  final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            // 调用了Application类中的makeApplication方法
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;
	....
	}

Application的makeApplication方法中创建了一个ContextIml

	 public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
	...
     // 可见这里新创建了ContextIml类，并传下去
	 ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
	...	
	}

最后在Instrumentation中的newApplication方法调用了attach方法关联

	 static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
 		// 调用了attach方法，把ContextIml关联
        app.attach(context);
        return app;
    }
 	
	//Application的attach方法
	/* package */ final void attach(Context context) {
		// 关联进去
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }

### 创建Activity关联Context ###

Activity创建在ActivityThread中调用

主要是调用了performLaunchActivity方法

	...
    // 先得到activity
	Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }

	...
		//如果Activity不为空，则创建了一个appContext
	 if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
	...
	//最后调用attach方法关联起来：
	    activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);
	}

关于createBaseContextForActivity方法：

	...
	//这里还是创建了一个ContextIml
	ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, r.token, displayId, r.overrideConfig);
        appContext.setOuterContext(activity);
        Context baseContext = appContext;
	...

### 创建Service关联Context ###

Service的创建也是调用了Activity中的handleCreateService的方法：

	private void handleCreateService(CreateServiceData data) {
	...
	//也是先找到这个Service
	 Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
	...
	// 最后进行创建ContextIml，然后attach起来
	    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            service.onCreate();
	...
	}

小结：

通过对ContextImp的分析可知，其方法的大多数操作都是直接调用其属性mPackageInfo(该属性类
型为PackageInfo)的相关方法而来。

这说明ContextImp是一种轻量级类，而PackageInfo才是真正重量级的类。而一个App里的
所有ContextIml实例，都对应同一个packageInfo对象。




