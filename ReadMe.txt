
摘自: http://weishu.me/2016/03/07/understand-plugin-framework-ams-pms-hook/

[[AMS获取过程]]
startActivity 是如何调用 AMS 最终通过 IPC 到 system_server 的.
两种形式: Context::startActivity 和 Activity::startActivity;

Context.startActivity:
public class Activity extends ContextThemeWrapper {}
public class ContextThemeWrapper extends ContextWrapper {}
public class ContextWrapper extends Context {}

Context::startActivity 是一个 abstract (抽象)函数;
// Context mBase; mBase 的真正实现是 ContextImpl 类;
public void ContextWrapper::startActivity(Intent intent) {
    mBase.startActivity(intent);
}

Context.startActivity 最终使用了 ContextImpl 里面的方法:
public void ContextImpl::startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();
    if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}

可以看到, 非 Activity 的 Context 里面启动 Activity 为什么需要添加 FLAG_ACTIVITY_NEW_TASK ;
真正的 startActivity 使用了 Instrumentation 类的 execStartActivity 方法, 继续跟踪:

// Instrumentation.java
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, String target,
    Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
可以发现真正调用的是 ActivityManagerNative.getDefault.startActivity 函数;

Activity.startActivity
// Activity.startActivity 最终调用了如下函数:
	Instrumentation.ActivityResult ar =
	    mInstrumentation.execStartActivity(
	        this, mMainThread.getApplicationThread(), mToken, this,
	        intent, requestCode, options);
通过对比跟踪,可以看到 Acitvity 和 ContextImpl 类启动 Activity 并无本质不同, 它们都是通过 
Instrumentation 这个辅助类 调用到 ActivityManagerNative 的方法;


[[ HOOK AMS ]]
我们现在知道, 其实 startActivity 最终通过 ActivityManagerNative 这个方法远程调用了 AMS 的
startActivity 方法;
ActivityManagerNative 实际上就是 ActivityManagerService 这个远程对象 的 Binder 代理对象;
每次需要与 AMS 打交道的时候, 需要借助这个代理对象通过驱动进而完成 IPC 调用;

我们看看 ActivityManagerNative 的 getDefault() 方法做了什么:
//Retrieve the system's default/global activity manager.
static public IActivityManager getDefault() {
    return gDefault.get();
}
gDefault 这个静态变量的定义如下:
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            IActivityManager am = asInterface(b);
            return am;
        }
    };
由于整个 Framework 与 AMS 打交道是如此频繁, framework 使用了一个单例把这个 AMS 的代理对象保存了起来;
这样只需要与 AMS 进行 IPC 调用, 获取这个单例即可。
这是 AMS 这个系统服务与其它普通服务的不同之处, 也是我们不通过 Binder Hook 的原因.
我们只需要简单滴 HOOK 掉这个单例即可。

对于 Android 不同版本之间, gDefault 这个单例的存储是不一样的, 使用 grepcode 进行比较:
( http://grepcode.com/file_/repository.grepcode.com/java/ext/com.google.android/android/4.0.1_r1/android/app/ActivityManagerNative.java/?v=diff&id2=2.3.3_r1 )
可以发现:
Android 2.x 系统直接使用了一个简单的 静态变量存储;
Android 4.x 以上抽象出了一个 Singleton 类;

我们以 4.x 以上的代码为例说明如何 HOOK 掉 AMS;
( 对于代理模式这个概念比较模糊的话,建议先看明白 <设计模式之蝉> 中有关代理模式这一章 )
    public static void hookActivityManager() {
        try {
            Class<?> activityManagerNativeClass = Class.forName("android.app.ActivityManagerNative");

            // 获取 gDefault 这个字段, 想办法替换它
            // Field的知识属于 Java 反射这块的知识;
            Field gDefaultField = activityManagerNativeClass.getDeclaredField("gDefault");
            gDefaultField.setAccessible(true);

            Object gDefault = gDefaultField.get(null);

            // 4.x以上的gDefault是一个 android.util.Singleton对象; 我们取出这个单例里面的字段
            Class<?> singleton = Class.forName("android.util.Singleton");
            Field mInstanceField = singleton.getDeclaredField("mInstance");
            mInstanceField.setAccessible(true);

            // ActivityManagerNative 的gDefault对象里面原始的 IActivityManager对象
            Object rawIActivityManager = mInstanceField.get(gDefault);

            // 创建一个这个对象的代理对象, 然后替换这个字段, 让我们的代理对象帮忙干活
            Class<?> iActivityManagerInterface = Class.forName("android.app.IActivityManager");
            Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                    new Class<?>[] { iActivityManagerInterface }, new HookHandler(rawIActivityManager));
            mInstanceField.set(gDefault, proxy);

        } catch (Exception e) {
            throw new RuntimeException("Hook Failed : ", e);
        }
    }
    // DroidPlugin 处理 AMS 的代码可以在 IActivityManagerHook 查看.

Android Framework 层对于四大组件的处理, 调用 AMS 服务的时候, 全部都是通过使用这种方式.
可以从 Context 类的 startActivity, startService, bindService, registerBroadcastReceiver, getContentResolver 等
入口进行跟踪, 最终都会发现它们都会使用 ActivityManagerNative 的这个 AMS 代理对象来完成对远程 AMS 的访问.


[[ PMS 获取过程 ]]
PMS 的获取也是通过 Context 完成的, 具体就是 getPackageManager 这个方法; 我们姑且当作已经之道了 Context 的实现在 ContextImpl 类
里面, 直奔 ContextImpl 类的 getPackageManager 方法:
	// ContextImpl 类:
    public PackageManager getPackageManager() {
        if (mPackageManager != null) {
            return mPackageManager;
        }

        IPackageManager pm = ActivityThread.getPackageManager();
        if (pm != null) {
            // Doesn't matter if we make more than one instance.
            return (mPackageManager = new ApplicationPackageManager(this, pm));
        }

        return null;
    }
可以看到, 这里干了两件事:
1.真正的 PMS 的代理对象在 ActivityThread 类里面;
2.ContextImpl 通过 ApplicationPackageManager 对它还进行了一层包装;

我们继续查看 ActivityThread 类的 getPackageManager 方法, 源码如下:
	// static IPackageManager sPackageManager;
    public static IPackageManager getPackageManager() {
        if (sPackageManager != null) {
            //Slog.v("PackageManager", "returning cur default = " + sPackageManager);
            return sPackageManager;
        }
        IBinder b = ServiceManager.getService("package");
        //Slog.v("PackageManager", "default service binder = " + b);
        sPackageManager = IPackageManager.Stub.asInterface(b);
        //Slog.v("PackageManager", "default service = " + sPackageManager);
        return sPackageManager;
    }
可以看到, 和 AMS 一样, PMS 的 Binder 代理对象也是一个全局变量存放在 一个静态字段 中;
我们可以 如法炮制, HOOK 掉 PMS;

现在我们的目的很明确, 如果需要 HOOK PMS 有两个地方需要 HOOK 掉:
1.ActivityThread 的静态字段 sPackageManager;
2.通过 Context 类的 getPackageManager 方法获取到的 ApplicationPackageManager 对象里面的 mPM 字段;

[[ HOOK PMS ]]
public static void hookPackageManager(Context context) {
	try {
	    // 获取全局的ActivityThread对象
	    Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
	    Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
	    Object currentActivityThread = currentActivityThreadMethod.invoke(null);

	    // 获取ActivityThread里面原始的 sPackageManager
	    Field sPackageManagerField = activityThreadClass.getDeclaredField("sPackageManager");
	    sPackageManagerField.setAccessible(true);
	    Object sPackageManager = sPackageManagerField.get(currentActivityThread);

	    // 准备好代理对象, 用来替换原始的对象
	    Class<?> iPackageManagerInterface = Class.forName("android.content.pm.IPackageManager");
	    Object proxy = Proxy.newProxyInstance(iPackageManagerInterface.getClassLoader(),
	            new Class<?>[] { iPackageManagerInterface },
	            new HookHandler(sPackageManager));

	    // 1. 替换掉ActivityThread里面的 sPackageManager 字段
	    sPackageManagerField.set(currentActivityThread, proxy);

	    // 2. 替换 ApplicationPackageManager里面的 mPm对象
	    PackageManager pm = context.getPackageManager();
	    Field mPmField = pm.getClass().getDeclaredField("mPM");
	    mPmField.setAccessible(true);
	    mPmField.set(pm, proxy);
	} catch (Exception e) {
	    throw new RuntimeException("hook failed", e);
	}
}
// DroidPlugin 处理 PMS 的代码可以在 IPackageManagerHook 查看.















































