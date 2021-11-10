---
layout:     post
title:      Android P 以上隐藏API的访问方法
subtitle:   谷歌从Android P 开始引入了针对非SDK接口（俗称为隐藏API）的使用限制。这是继 Android N上针对NDK中私有库的链接限制之后的又一次重大调整。
date:       2021-08-12
author:     coderman
header-img: img/article-bg.jpg
top: false
catalog: true 
tags:
    - Android
    - API访问
---


# 一，前言
谷歌从Android P 开始引入了针对非SDK 接口（俗称为隐藏API）的使用限制。这是继 Android N上针对NDK中私有库的链接限制之后的又一次重大调整。从今以后，不论是native层的NDK还是 Java层的SDK，我们只能使用Google提供的、公开的标准接口。这对开发者以及用户乃至整个Android生态，当然是一件好事。但这也同时意味着Android上的各种黑科技有可能会逐渐走向消亡。

作为一个有追求的开发者，我们既要尊重并遵守规则，也要有能力在必要的时候突破规则的束缚，带着镣铐跳舞。那么今天就来探讨一下，如何突破Android P以上针对非SDK接口调用的限制。

系统是如何实现这个限制的？知己知彼，百战不殆。既然我们想要突破这个限制，自然先得弄清楚，系统是如何给我们施加这个限制的。
# 二，源码分析
官方文档 中说，通过反射或者JNI访问非公开接口时会触发警告/异常等，那么不妨跟踪一下反射的流程，看看系统到底在哪一步做的限制（以下的源码分析大可以走马观花的看一下，需要的时候自己再仔细看）。
我们从 java.lang.Class.getDeclaredMethod(String) 看起，这个方法在Java层最终调用到了 getDeclaredMethodInternal 这个native方法，看一下这个方法的源码：

<pre><code>
static jobject Class_getDeclaredMethodInternal(JNIEnv* env, jobject javaThis,
                                               jstring name, jobjectArray args) {
  ScopedFastNativeObjectAccess soa(env);
  StackHandleScope&lt;1&gt; hs(soa.Self());
  DCHECK_EQ(Runtime::Current()-&gt;GetClassLinker()-&gt;GetImagePointerSize(), kRuntimePointerSize);
  DCHECK(!Runtime::Current()-&gt;IsActiveTransaction());
  Handle&lt;mirror::Method&gt; result = hs.NewHandle(
      mirror::Class::GetDeclaredMethodInternal&lt;kRuntimePointerSize, false&gt;(
          soa.Self(),
          DecodeClass(soa, javaThis),
          soa.Decode&lt;mirror::String&gt;(name),
          soa.Decode&lt;mirror::ObjectArray&lt;mirror::Class&gt;&gt;(args)));
  if (result == nullptr || ShouldBlockAccessToMember(result-&gt;GetArtMethod(), soa.Self())) {
    return nullptr;
  }
  return soa.AddLocalReference&lt;jobject&gt;(result.Get());
}
</code></pre>
注意到那个`ShouldBlockAccessToMember` (在Q以上的版本这个方法名称改为了`ShouldDenyAccessToMember`)调用了吗？如果它返回false，那么直接返回nullptr，上层就会抛 NoSuchMethodXXX 异常；也就触发系统的限制了。于是我们继续跟踪这个方法，这个方法的实现在 java_lang_Class.cc，源码如下：
<pre><code>
ALWAYS_INLINE static bool ShouldBlockAccessToMember(T* member, Thread* self)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  hiddenapi::Action action = hiddenapi::GetMemberAction(
      member, self, IsCallerTrusted, hiddenapi::kReflection);
  if (action != hiddenapi::kAllow) {
    hiddenapi::NotifyHiddenApiListener(member);
  }
  return action == hiddenapi::kDeny;
}
</code></pre>
毫无疑问，我们应该继续看 hidden_api.cc 里面的 GetMemberAction方法 ：
<pre><code>
template&lt;typename T&gt;
inline Action GetMemberAction(T* member,
                              Thread* self,
                              std::function&lt;bool(Thread*)&gt; fn_caller_is_trusted,
                              AccessMethod access_method)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK(member != nullptr);
  // Decode hidden API access flags.
  // NB Multiple threads might try to access (and overwrite) these simultaneously,
  // causing a race. We only do that if access has not been denied, so the race
  // cannot change Java semantics. We should, however, decode the access flags
  // once and use it throughout this function, otherwise we may get inconsistent
  // results, e.g. print whitelist warnings (b/78327881).
  HiddenApiAccessFlags::ApiList api_list = member-&gt;GetHiddenApiAccessFlags();
  Action action = GetActionFromAccessFlags(member-&gt;GetHiddenApiAccessFlags());
  if (action == kAllow) {
    // Nothing to do.
    return action;
  }
  // Member is hidden. Invoke `fn_caller_in_platform` and find the origin of the access.
  // This can be *very* expensive. Save it for last.
  if (fn_caller_is_trusted(self)) {
    // Caller is trusted. Exit.
    return kAllow;
  }
  // Member is hidden and caller is not in the platform.
  return detail::GetMemberActionImpl(member, api_list, action, access_method);
}
</code></pre>
可以看到，关键来了。此方法有三个return语句，如果我们能干涉这几个语句的返回值，那么就能影响到系统对隐藏API的判断；进而欺骗系统，绕过限制。
# 三，访问原理
从源码中我们可以看到有一个 `fn_caller_is_trusted`的条件：如果调用者是系统类，那么就允许被调用。这是显而易见的，毕竟这些私有 API 就是给系统用的，如果系统自己都被拒绝了，那还玩什么呢？
也就是说，如果我们能以系统类的身份去反射，那么就能畅通无阻。问题是，我们如何以「系统的身份去反射」呢？一种最常见的办法是，我们自己写一个类，然后通过某种途径把这个类的 ClassLoader 设置为系统的 ClassLoader，再借助这个类去反射其他类。但是这里的「通过某种途径」依然要使用一些黑科技才能实现，过程比较麻烦。

以系统类的身份去反射 有两个意思，1. 直接把我们自己变成系统类；2. 借助系统类去调用反射。我们一个个分析。
「直接把我们自己变成系统类」这个方式有童鞋可能觉得天方夜谭，APP 的类怎么可能成为系统类？但是，一定不要被自己的固有思维给局限，一切皆有可能！我们知道，对APP来说，所谓的系统类就是被 BootstrapClassLoader 加载的类，这个 ClassLoader 并非普通的 DexClassLoader，因此我们无法通过插入 dex path的方式注入类。但是，Android 的 ART 在 Android O 上引入了 JVMTI，JVMTI 提供了将某一个类转换为 BootstrapClassLoader 中的类的方法！具体来说，我们写一个类暴露反射相关的接口，然后通过 JVMTI 提供的 AddToBootstrapClassLoaderSearch将此类加入 BootstrapClassLoader 就实现目的了。不过，JVMTI 要在 release 版本的 APP 上运行依然需要 Hack，所以这种途径与其他的黑科技无本质区别。

第二种方法，「借助系统的类去反射」也就是说，如果系统有一个方法systemMethod，这个systemMethod 去调用反射相反的方法，那么systemMethod毋庸置疑会反射成功。但是，我们从哪去找到这么一个方法给我们用？事实上，我们不仅能找到这样的方法，而且这个方法能帮助我们调用任意的函数，那就是反射本身！可能你已经绕晕了，我解释一下：
首先，我们通过反射 API 拿到 getDeclaredMethod 方法。getDeclaredMethod 是 public 的，不存在问题；这个通过反射拿到的方法我们称之为元反射方法。然后，我们通过刚刚反射拿到元反射方法去反射调用 getDeclardMethod。这里我们就实现了以系统身份去反射的目的——反射相关的 API 都是系统类，因此我们的元反射方法也是被系统类加载的方法；所以我们的元反射方法调用的 getDeclardMethod 会被认为是系统调用的，可以反射任意的方法。
伪代码如下：
<pre><code>
Method metaGetDeclaredMethod =
        Class.class.getDeclaredMethod("getDeclardMethod"); // 公开API，无问题
Method hiddenMethod = metaGetDeclaredMethod.invoke(hiddenClass,
        "hiddenMethod", "hiddenMethod参数列表"); // 系统类通过反射使用隐藏 API，检查直接通过。
hiddenMethod.invoke // 正确找到 Method 直接反射调用
</code></pre>
到这里，我们已经能通过「元反射」的方式去任意获取隐藏方法或者隐藏 Field 了。但是，如果我们所有使用的隐藏方法都要这么干，那有点太麻烦了。实际上我们在仔细阅读源码就会发现，隐藏 API 调用还有「豁免」条件，具体代码如下
<pre><code>
if (shouldWarn || action == kDeny) {
    if (member_signature.IsExempted(runtime->GetHiddenApiExemptions())) {
      action = kAllow;
      // Avoid re-examining the exemption list next time.
      // Note this results in no warning for the member, which seems like what one would expect.
      // Exemptions effectively adds new members to the whitelist.
      MaybeWhitelistMember(runtime, member);
      return kAllow;
    }
    // 略    
}
</code></pre>
只要 `IsExempted` 方法返回 true，就算这个方法在黑名单中，依然会被放行然后允许被调用。我们再观察一下IsExempted方法：
<pre><code>
bool MemberSignature::IsExempted(const std::vector&lt;std::string&gt;& exemptions) {
  for (const std::string& exemption : exemptions) {
    if (DoesPrefixMatch(exemption)) {
      return true;
    }
  }
  return false;
}
</code></pre>

继续跟踪传递进来的参数` runtime->GetHiddenApiExemptions()` 会有个有趣的发现：这个API 竟然是暴露到 Java 层的，有一个对应的 ` VMRuntime.setHiddenApiExemptions`  Java方法；也就是说，只要我们通过 ` VMRuntime.setHiddenApiExemptions`  设置下豁免条件，我们就能愉快滴使用反射了。
再结合上面这个方法，我们只需要通过 「元反射」来反射调用 VMRuntime.setHiddenApiExemptions 就能将我们自己要使用的隐藏 API 全部都豁免掉了。更进一步，如果我们再观察下上面的 IsExempted 方法里面调用的 DoesPrefixMatch，发现这玩意儿在对方法签名进行前缀匹配；童鞋们，我们所有Java方法类的签名都是以 L开头啊！我们可以直接传个 L进去，所有的隐藏API就全部被赦免了！
# 四，访问方案
下面给出最终的访问解决方案，以供参考
<pre><code>
public final class ReflectionUtils{
    private static Object sVMRuntime;
    private static Method setHiddenApiExemptions;

    static {
        int mV = Build.VERSION.SDK_INT;
        if (mV >= 29) {
            try {
                Method forName = Class.class.getDeclaredMethod("forName", String.class);
                Method getDeclaredMethod = Class.class.getDeclaredMethod("getDeclaredMethod", String.class, Class[].class);
                Class<?> vmRuntimeClass = (Class<?>) forName.invoke(null, "dalvik.system.VMRuntime");
                Method getRuntime = (Method) getDeclaredMethod.invoke(vmRuntimeClass, "getRuntime", null);
                setHiddenApiExemptions = (Method) getDeclaredMethod.invoke(vmRuntimeClass, "setHiddenApiExemptions", new Class[]{String[].class});
                setHiddenApiExemptions.setAccessible(true);
                sVMRuntime = getRuntime.invoke(null);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    
    //绕过限制
    public static boolean doReflection() {
        if (sVMRuntime == null || setHiddenApiExemptions == null) {
            return false;
        }
        try {
            setHiddenApiExemptions.invoke(sVMRuntime, new Object[]{new String[]{"L"}});
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }
}
</code></pre>
