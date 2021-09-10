---
layout:     post
title:      Android10 PKMS相关类分析
subtitle:   本文罗列学习一下与PackageManagerService相关的一些类
date:       2020-05-30
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - Android10
    - Android
    - PKMS
    - 系统服务
---
<h2 id="1-Settings类"><a href="#1-Settings类" class="headerlink" title="1.Settings类"></a>1.Settings类</h2>
<pre><code>
// Settins文件 data/system/packages.xml
private final File mSettingsFilename;

//这个文件不一定存在，是备份文件，如果存在则说明更新packages.xml出错
//data/system/packages_backup.xml
private final File mBackupSettingsFilename;

//data/system/packages.list
private final File mPackageListFilename;

// key是包名，PackageSetting主要包含app的基本信息，如安装位置，lib位置等
 /** Map from package name to settings */
final ArrayMap<String, PackageSetting> mPackages = new ArrayMap<>();

/*key是类似“android.ui.system”这样的字段，在Android中每个应用都有一个UID，两个相同的UID的应用可以运行在/一个进程中，为了让两个应用运行在一个进程中，需要在manifest中设置sharedUserId这个属性，这个属性是字符串，但是在linux系统中uid是一个整型，因此就有了SharedUserSetting类型，这个类型除了name还有uid(对应linux中的uid),还有一个列表字段，用于记录系统中相同shardUserId的应用。*/
final ArrayMap<String, SharedUserSetting> mSharedUsers =
      new ArrayMap<String, SharedUserSetting>();
   
/*主要保存的是/system/etc/permissions/platform.xml中的permission标签内容，因为Android系统是基于linux的系统，有用户组的概念，platform定义了一些权限，并且定制了哪些用户组具有哪些权限，一旦应用属于某个用户组，那么它就有这个用户组的所有权限*/
final PermissionSettings mPermissions;

</code></pre>
<h3 id="1-1-Setting构造函数"><a href="#1-1-Setting构造函数" class="headerlink" title="1.1 Setting构造函数"></a>1.1 Setting构造函数</h3>
<pre><code>
Settings(File dataDir, PermissionSettings permission, Object lock) {
       mLock = lock;
       mPermissions = permission;
       //创建mRuntimePermissionsPersistence，是Setting内部类
       mRuntimePermissionsPersistence = new RuntimePermissionPersistence(mLock);
       
       //初始化文件路径
       mSystemDir = new File(dataDir, "system");
       mSystemDir.mkdirs();
       FileUtils.setPermissions(mSystemDir.toString(),
               FileUtils.S_IRWXU|FileUtils.S_IRWXG
               |FileUtils.S_IROTH|FileUtils.S_IXOTH,
               -1, -1);
       mSettingsFilename = new File(mSystemDir, "packages.xml");
       mBackupSettingsFilename = new File(mSystemDir, "packages-backup.xml");
       mPackageListFilename = new File(mSystemDir, "packages.list");
       FileUtils.setPermissions(mPackageListFilename, 0640, SYSTEM_UID, PACKAGE_INFO_GID);

       final File kernelDir = new File("/config/sdcardfs");
       mKernelMappingFilename = kernelDir.exists() ? kernelDir : null;
       
       //下面两个文件路径不推荐使用
       // Deprecated: Needed for migration
       mStoppedPackagesFilename = new File(mSystemDir, "packages-stopped.xml");
       mBackupStoppedPackagesFilename = new File(mSystemDir, "packages-stopped-backup.xml");
   }

</code></pre>
<p>Setting构造函数主要工作是创建系统文件夹，一些包管理的文件</p>
<p>packages.xml、packages-backup.xml是一组，用于描述系统所安装的Package信息，其中packages-backup.xml是packages.xml的备份</p>
<p>packages.list用于描述系统中存在的所有非系统自带的apk信息以及UID大于10000的apk。当APK有变化时，PKMS就会更新该文件。</p>
<h3 id="1-2-addSharedUserLPw方法"><a href="#1-2-addSharedUserLPw方法" class="headerlink" title="1.2 addSharedUserLPw方法"></a>1.2 addSharedUserLPw方法</h3><p>该方法将shareUserId name和一个int类型的UID对应起来。UID的定义在Process.java中。</p>
<pre><code>
SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
      //获取SharedUserSetting对象
      SharedUserSetting s = mSharedUsers.get(name);
      if (s != null) {
          if (s.userId == uid) {
              return s;
          }
          PackageManagerService.reportSettingsProblem(Log.ERROR,
                  "Adding duplicate shared user, keeping first: " + name);
          return null;
      }
      //没有在mSharedUsers找到则新建，并保存起来
      s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
      s.userId = uid;
      if (addUserIdLPw(uid, s, name)) {
          mSharedUsers.put(name, s);
          return s;
      }
      return null;
  }

</code></pre>
<pre><code>
/**
 * Defines the root UID.
 */
public static final int ROOT_UID = 0;

/**
 * Defines the UID/GID under which system code runs.
 */
public static final int SYSTEM_UID = 1000;

/**
 * Defines the UID/GID under which the telephony code runs.
 */
public static final int PHONE_UID = 1001;

/**
 * Defines the UID/GID for the user shell.
 */
public static final int SHELL_UID = 2000;

/**
 * Defines the UID/GID for the log group.
 * @hide
 */
@UnsupportedAppUsage
public static final int LOG_UID = 1007;

/**
 * Defines the UID/GID for the WIFI supplicant process.
 * @hide
 */
@UnsupportedAppUsage
public static final int WIFI_UID = 1010;
...

//第一个应用package的起始UID为10000
 /**
 * Defines the start of a range of UIDs (and GIDs), going from this
 * number to {@link #LAST_APPLICATION_UID} that are reserved for assigning
 * to applications.
 */
public static final int FIRST_APPLICATION_UID = 10000;
//最后一个应用package的UID为19999
/**
 * Last of application-specific UIDs starting at
 * {@link #FIRST_APPLICATION_UID}.
 */
public static final int LAST_APPLICATION_UID = 19999;

</code></pre>
<p>Setting模块的AndroidManifest.xml里面，如下所示：</p>
<pre><code>
&lt;manifest xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:androidprv="http://schemas.android.com/apk/prv/res/android"
        package="com.android.settings"
        coreApp="true"
        android:sharedUserId="android.uid.system"&gt;

</code></pre>
<p> 在xml里面android:sharedUserId属性设置为”android.uid.system”。sharedUserId这个属性主要有两个作用：</p>
<p>1.两个或者多个声明了同一种sharedUserId的应用可以共享彼此的数据</p>
<p>2.通过声明特定sharedUserId，该应用所在进程将赋予指定的UID。如Setting声明了system的uid，则就可以共享system用户所对应的权限。</p>
<p>除了了xml了声明sharedUserId外，应用编译的时候还必须使用对应的证书进行签名。如Setting需要platform的签名。</p>
<h3 id="1-3-PID、UID、GID的区别"><a href="#1-3-PID、UID、GID的区别" class="headerlink" title="1.3 PID、UID、GID的区别"></a>1.3 PID、UID、GID的区别</h3><p>PID是进程的身份识别，程序一旦运行，就会给应用分配唯一的PID。一个应用可能包含多个进程，每个进程有唯一的一个PID。进程终止后PID被系统回收，再次打开应用，会分配一个PID（新进程的PID一般比之前的号大）。</p>
<p>调用adb shell ps可以查看系统运行的进程。</p>
<p>UID是用户ID。UID在linux中就是用户ID,表明哪个用户运行了这个程序，主要用于权限的管理。而Android为单用户系统，这时候UID被赋予了新的使命，数据共享，为了实现数据共享，android为每个应用都分配了不同的uid,不像传统的linux，每个用户相同就为之分配相同的UID。</p>
<p>GID时用户组ID。对于普通的应用程序来说GID等于UID，由于每个应用程序的UID和GID不相同，所以不管是native还是java层都能够达到保护私有数据的作用。</p>
<p>adb shell cat /proc/PID号/status</p>
<p>SharedUserSetting类架构</p>
<p><img src="/2019/PKMS相关类分析/SharedUserSetting.png" alt=""></p>
<p>PKMS的构造函数创建一个Settings的实例mSettings，mSettings有三个成员变量mSharedUsers，mUserIds，mOtherUserIds。addSharedUserLPw方法都涉及这三个成员变量。SharedUserSetting的成员变量packages是一个PackageSetting类型的ArraySet。PackageSetting继承自PackageSettingBase，PackageSetting保存着package的多种信息。</p>
<p><img src="/2019/PKMS相关类分析/SharedUserId.png" alt=""></p>
<h2 id="2-SystemConfig类"><a href="#2-SystemConfig类" class="headerlink" title="2.SystemConfig类"></a>2.SystemConfig类</h2><h3 id="2-1-SystemConfig构造函数"><a href="#2-1-SystemConfig构造函数" class="headerlink" title="2.1 SystemConfig构造函数"></a>2.1 SystemConfig构造函数</h3><p>主要读取下面路径的配置</p>
<p>/system/etc/、/system/etc/、/vendor/etc、/odm/etc、/oem/etc、/product/etc 目录下sysconfig和permissions</p>
<pre><code>
SystemConfig() {
       // Read configuration from system
       readPermissions(Environment.buildPath(
               Environment.getRootDirectory(), "etc", "sysconfig"), ALLOW_ALL);

       // Read configuration from the old permissions dir
       readPermissions(Environment.buildPath(
               Environment.getRootDirectory(), "etc", "permissions"), ALLOW_ALL);

       // Vendors are only allowed to customze libs, features and privapp permissions
       int vendorPermissionFlag = ALLOW_LIBS | ALLOW_FEATURES | ALLOW_PRIVAPP_PERMISSIONS;
       if (Build.VERSION.FIRST_SDK_INT <= Build.VERSION_CODES.O_MR1) {
           // For backward compatibility
           vendorPermissionFlag |= (ALLOW_PERMISSIONS | ALLOW_APP_CONFIGS);
       }
       readPermissions(Environment.buildPath(
               Environment.getVendorDirectory(), "etc", "sysconfig"), vendorPermissionFlag);
       readPermissions(Environment.buildPath(
               Environment.getVendorDirectory(), "etc", "permissions"), vendorPermissionFlag);

       // Allow ODM to customize system configs as much as Vendor, because /odm is another
       // vendor partition other than /vendor.
       int odmPermissionFlag = vendorPermissionFlag;
       readPermissions(Environment.buildPath(
               Environment.getOdmDirectory(), "etc", "sysconfig"), odmPermissionFlag);
       readPermissions(Environment.buildPath(
               Environment.getOdmDirectory(), "etc", "permissions"), odmPermissionFlag);

       String skuProperty = SystemProperties.get(SKU_PROPERTY, "");
       if (!skuProperty.isEmpty()) {
           String skuDir = "sku_" + skuProperty;

           readPermissions(Environment.buildPath(
                   Environment.getOdmDirectory(), "etc", "sysconfig", skuDir), odmPermissionFlag);
           readPermissions(Environment.buildPath(
                   Environment.getOdmDirectory(), "etc", "permissions", skuDir),
                   odmPermissionFlag);
       }

       // Allow OEM to customize features and OEM permissions
       int oemPermissionFlag = ALLOW_FEATURES | ALLOW_OEM_PERMISSIONS;
       readPermissions(Environment.buildPath(
               Environment.getOemDirectory(), "etc", "sysconfig"), oemPermissionFlag);
       readPermissions(Environment.buildPath(
               Environment.getOemDirectory(), "etc", "permissions"), oemPermissionFlag);

       // Allow Product to customize all system configs
       readPermissions(Environment.buildPath(
               Environment.getProductDirectory(), "etc", "sysconfig"), ALLOW_ALL);
       readPermissions(Environment.buildPath(
               Environment.getProductDirectory(), "etc", "permissions"), ALLOW_ALL);
   }

</code></pre>
<p>SystemConfig构造函数中主要通过readPermissions函数将对应目录下的xml文件中定义的各个节点读取出来保存到SystemConfig成员变量中。在终端的/system/etc/permissions目录下可以看到很多xml配置文件，如下：</p>
<pre><code>
HWSTF:/system/etc/permissions $ ls -all
total 300
drwxr-xr-x  2 root root  4096 2018-08-08 00:01:00.000000000 +0800 .
drwxr-xr-x 26 root root  4096 2018-08-08 00:01:00.000000000 +0800 ..
-rw-r--r--  1 root root  1515 2018-08-08 00:01:00.000000000 +0800 HiViewTunnel-core.xml
-rw-r--r--  1 root root   830 2018-08-08 00:01:00.000000000 +0800 android.hardware.bluetooth_le.xml
-rw-r--r--  1 root root   927 2018-08-08 00:01:00.000000000 +0800 android.hardware.faketouch.xml
-rw-r--r--  1 root root   834 2018-08-08 00:01:00.000000000 +0800 android.hardware.fingerprint.xml
-rw-r--r--  1 root root   942 2018-08-08 00:01:00.000000000 +0800 android.hardware.location.gps.xml
-rw-r--r--  1 root root   949 2018-08-08 00:01:00.000000000 +0800 android.hardware.location.xml
-rw-r--r--  1 root root   888 2018-08-08 00:01:00.000000000 +0800 android.hardware.nfc.hce.xml
-rw-r--r--  1 root root   891 2018-08-08 00:01:00.000000000 +0800 android.hardware.nfc.hcef.xml
-rw-r--r--  1 root root   921 2018-08-08 00:01:00.000000000 +0800 android.hardware.nfc.xml
-rw-r--r--  1 root root   870 2018-08-08 00:01:00.000000000 +0800 android.hardware.opengles.aep.xml
-rw-r--r--  1 root root   824 2018-08-08 00:01:00.000000000 +0800 
...

</code></pre>
<p>这些配置文件都是编译时从framework指定位置拷贝过来的（framework/native/data/etc）</p>
<p>readPermissions方法内部调用readPermissionsFromXml方法来解析xml里面的各个节点，其中xml涉及到的标签内容有permission、assign-permission、library、feature等，这些标签的内容解析出来保存到SystemConfig的对应数据结构的全局变量中，以便管理查询。</p>
<p>feature用来描述设备是否支持硬件特性；  library用于指定系统库，当应用程序运行时，系统会为进城加载一些必须的库； assign-permission将system中描述的permission与uid关联； permission将permission和gid关联。</p>
<p>总结下SystemConfig初始化时解析xml文件节点以及对应的全局变量。</p>
<p><img src="/2019/PKMS相关类分析/systemconfig.png" alt=""></p>
<h2 id="3-PackageParser"><a href="#3-PackageParser" class="headerlink" title="3. PackageParser"></a>3. PackageParser</h2><p>这个类作用是解析APK，在其类中注释如下：</p>
<pre><code>
/**
 * Parser for package files (APKs) on disk. This supports apps packaged either
 * as a single "monolithic" APK, or apps packaged as a "cluster" of multiple
 * APKs in a single directory.
 * 
 * Apps packaged as multiple APKs always consist of a single "base" APK (with a
 * {@code null} split name) and zero or more "split" APKs (with unique split
 * names). Any subset of those split APKs are a valid install, as long as the
 * following constraints are met:
 * 
 * All APKs must have the exact same package name, version code, and signing
 * certificates.
 * All APKs must have unique split names.
 * All installations must contain a single base APK.
 * 
 *
 * @hide
 */

</code></pre>
<p>这个类主要用于解析apk安装包，它能解析单一apk文件，也能够解析multiple APKs（一个apk文件里面包含多个apk文件）。这些multiple APKs需要满足下面几个条件：</p>
<p>1.所有的apk必须具有完全相同的软件包包名，版本代码和签名证书</p>
<p>2.所有的apk必须具有唯一的拆分名称</p>
<p>3.所有安装必须含有一个单一的apk</p>
<p>解析步骤</p>
<p>1.将apk解析成package</p>
<p>2.将package转化为packageinfo</p>
<p>类结构</p>
<p>里面有很多内部类和方法，下面讲介绍里面的主要内部类以及部分解析的方法</p>
<h3 id="3-1-内部类"><a href="#3-1-内部类" class="headerlink" title="3.1 内部类"></a>3.1 内部类</h3><h4 id="3-1-1-NewPermissionInfo"><a href="#3-1-1-NewPermissionInfo" class="headerlink" title="3.1.1 NewPermissionInfo"></a>3.1.1 NewPermissionInfo</h4><p>记录新的权限</p>
<pre><code>
/** @hide */
   public static class NewPermissionInfo {
       //权限名称
       @UnsupportedAppUsage
       public final String name;
       @UnsupportedAppUsage
       //权限的开始版本号
       public final int sdkVersion;
       //文件的版本号，一般为0
       public final int fileVersion;

       public NewPermissionInfo(String name, int sdkVersion, int fileVersion) {
           this.name = name;
           this.sdkVersion = sdkVersion;
           this.fileVersion = fileVersion;
       }
   }

</code></pre>
<h4 id="3-1-2-SplitPermissionInfo"><a href="#3-1-2-SplitPermissionInfo" class="headerlink" title="3.1.2  SplitPermissionInfo"></a>3.1.2  SplitPermissionInfo</h4><p>主要记录一个权限拆分为颗粒度更小的权限</p>
<pre><code>
/** @hide */
   public static class SplitPermissionInfo {
       //表示旧的权限
       public final String rootPerm;
       表示旧的权限拆分为颗粒度更小的权限
       //public final String[] newPerms;
       表示在那个版本上拆分
       //public final int targetSdk;

       public SplitPermissionInfo(String rootPerm, String[] newPerms, int targetSdk) {
           this.rootPerm = rootPerm;
           this.newPerms = newPerms;
           this.targetSdk = targetSdk;
       }
   }

</code></pre>
<h4 id="3-1-3-ParsePackageItemArgs"><a href="#3-1-3-ParsePackageItemArgs" class="headerlink" title="3.1.3  ParsePackageItemArgs"></a>3.1.3  ParsePackageItemArgs</h4><p>主要为解析包单个item的参数</p>
<pre><code>
static class ParsePackageItemArgs {
        //表示安装包的包对象package
        final Package owner;
        //表示错误信息
        final String[] outError;
        //表示安装包中名字对应的资源id
        final int nameRes;
        //表示安装包中label对应的资源id
        final int labelRes;
        //表示安装包中icon对应的资源id
        final int iconRes;
        //表示安装包中roundIcon对应的资源id
        final int roundIconRes;
        //表示安装包中logo对应的资源id
        final int logoRes;
        //表示安装包中banner对应的资源id
        final int bannerRes;

        String tag;
        TypedArray sa;

        ParsePackageItemArgs(Package _owner, String[] _outError,
                int _nameRes, int _labelRes, int _iconRes, int _roundIconRes, int _logoRes,
                int _bannerRes) {
            owner = _owner;
            outError = _outError;
            nameRes = _nameRes;
            labelRes = _labelRes;
            iconRes = _iconRes;
            logoRes = _logoRes;
            bannerRes = _bannerRes;
            roundIconRes = _roundIconRes;
        }
    }

</code></pre>
<h4 id="3-1-4-ParseComponentArgs"><a href="#3-1-4-ParseComponentArgs" class="headerlink" title="3.1.4  ParseComponentArgs"></a>3.1.4  ParseComponentArgs</h4><p>主要为解析包中单个组件的参数</p>
<pre><code>
/** @hide */
   @VisibleForTesting
   public static class ParseComponentArgs extends ParsePackageItemArgs {
       //表示该组件对应的进程，如果设置独立进程则表示为独立进程的名字
       final String[] sepProcesses;
       //表示组件对应的进程的资源id
       final int processRes;
        //表示组件对应的进程的描述id
       final int descriptionRes;
       //表示组件是否可用
       final int enabledRes;
       //表示该组件的标志位
       int flags;

       public ParseComponentArgs(Package _owner, String[] _outError,
               int _nameRes, int _labelRes, int _iconRes, int _roundIconRes, int _logoRes,
               int _bannerRes,
               String[] _sepProcesses, int _processRes,
               int _descriptionRes, int _enabledRes) {
           super(_owner, _outError, _nameRes, _labelRes, _iconRes, _roundIconRes, _logoRes,
                   _bannerRes);
           sepProcesses = _sepProcesses;
           processRes = _processRes;
           descriptionRes = _descriptionRes;
           enabledRes = _enabledRes;
       }
   }

</code></pre>
<h4 id="3-1-5-PackageLite"><a href="#3-1-5-PackageLite" class="headerlink" title="3.1.5  PackageLite"></a>3.1.5  PackageLite</h4><p>表示在解析过程中的一个轻量级的独立的安装包</p>
<pre><code>
/**
    * Lightweight parsed details about a single package.
    */
   public static class PackageLite {
        //表示包名
       @UnsupportedAppUsage
       public final String packageName;
       //表示版本号
       public final int versionCode;
       //表示主版本号
       public final int versionCodeMajor;
       @UnsupportedAppUsage
       //表示安装位置的属性，可以有几个常量的选择，比如PackageInfo.INSTALL_LOACTION_AUTO
       public final int installLocation;
       //表示验证对象
       public final VerifierInfo[] verifiers;

       //如果有拆包，则表示拆包的名字数组
       /** Names of any split APKs, ordered by parsed splitName */
       public final String[] splitNames;
       
       //拆包是否有feature
       /** Names of any split APKs that are features. Ordered by splitName */
       public final boolean[] isFeatureSplits;
      
       //拆包的uses和config
       /** Dependencies of any split APKs, ordered by parsed splitName */
       public final String[] usesSplitNames;
       public final String[] configForSplit;

       //表示代码的路径，单个apk对应的是base apk路径，集群apk则是集群apk的目录
       /**
        * Path where this package was found on disk. For monolithic packages
        * this is path to single base APK file; for cluster packages this is
        * path to the cluster directory.
        */
       public final String codePath;
       //base apk的路径
       /** Path of base APK */
       public final String baseCodePath;
       //拆分apk的路径
       /** Paths of any split APKs, ordered by parsed splitName */
       public final String[] splitCodePaths;
       //base apk的调整版本号   
       /** Revision code of base APK */
       public final int baseRevisionCode;
       //拆分apk的调整版本号  
       /** Revision codes of any split APKs, ordered by parsed splitName */
       public final int[] splitRevisionCodes;
       //是不是核心app
       public final boolean coreApp;
       //是不是debug
       public final boolean debuggable;
       //是不是支持多平台，主要是指cpu平台
       public final boolean multiArch;
       //是不是用32位的so库
       public final boolean use32bitAbi;
       //是否需要提取so库
       public final boolean extractNativeLibs;
       //拆分包是否是独立
       public final boolean isolatedSplits;

       public PackageLite(String codePath, ApkLite baseApk, String[] splitNames,
               boolean[] isFeatureSplits, String[] usesSplitNames, String[] configForSplit,
               String[] splitCodePaths, int[] splitRevisionCodes) {
           this.packageName = baseApk.packageName;
           this.versionCode = baseApk.versionCode;
           this.versionCodeMajor = baseApk.versionCodeMajor;
           this.installLocation = baseApk.installLocation;
           this.verifiers = baseApk.verifiers;
           this.splitNames = splitNames;
           this.isFeatureSplits = isFeatureSplits;
           this.usesSplitNames = usesSplitNames;
           this.configForSplit = configForSplit;
           this.codePath = codePath;
           this.baseCodePath = baseApk.codePath;
           this.splitCodePaths = splitCodePaths;
           this.baseRevisionCode = baseApk.revisionCode;
           this.splitRevisionCodes = splitRevisionCodes;
           this.coreApp = baseApk.coreApp;
           this.debuggable = baseApk.debuggable;
           this.multiArch = baseApk.multiArch;
           this.use32bitAbi = baseApk.use32bitAbi;
           this.extractNativeLibs = baseApk.extractNativeLibs;
           this.isolatedSplits = baseApk.isolatedSplits;
       }
</code></pre>
<h4 id="3-1-6-ApkLite"><a href="#3-1-6-ApkLite" class="headerlink" title="3.1.6  ApkLite"></a>3.1.6  ApkLite</h4>
<p>表示解析过程中的一个轻量级独立的apk</p>
<pre><code>
/**
    * Lightweight parsed details about a single APK file.
    */
   public static class ApkLite {
       //表示代码的路径
       public final String codePath;
       //表示包名
       public final String packageName;
       //表示拆分的包名
       public final String splitName;
        //表示拆包是否有Feature
       public boolean isFeatureSplit;
        //表示拆包的配置
       public final String configForSplit;
        //表示拆包名字的uses
       public final String usesSplitName;
        //表示版本号
       public final int versionCode;
        //表示主版本号
       public final int versionCodeMajor;
       //表示调整的版本号
       public final int revisionCode;
       //表示安装位置的属性，可以有几个常量的选择，比如PackageInfo.INSTALL_LOACTION_AUTO
       public final int installLocation;
       //表示验证对象
       public final VerifierInfo[] verifiers;
       //表示签名对象
       public final SigningDetails signingDetails;
       //是不是核心app
       public final boolean coreApp;
        //是不是debug
       public final boolean debuggable;
        //是不是支持多平台，主要是指cpu平台
       public final boolean multiArch;
        //是不是用32位的so库
       public final boolean use32bitAbi;
        //是否需要提取so库
       public final boolean extractNativeLibs;
        //拆分包是否是独立
       public final boolean isolatedSplits;

       public ApkLite(String codePath, String packageName, String splitName,
               boolean isFeatureSplit,
               String configForSplit, String usesSplitName, int versionCode, int versionCodeMajor,
               int revisionCode, int installLocation, List&lt;VerifierInfo&gt; verifiers,
               SigningDetails signingDetails, boolean coreApp,
               boolean debuggable, boolean multiArch, boolean use32bitAbi,
               boolean extractNativeLibs, boolean isolatedSplits) {
           this.codePath = codePath;
           this.packageName = packageName;
           this.splitName = splitName;
           this.isFeatureSplit = isFeatureSplit;
           this.configForSplit = configForSplit;
           this.usesSplitName = usesSplitName;
           this.versionCode = versionCode;
           this.versionCodeMajor = versionCodeMajor;
           this.revisionCode = revisionCode;
           this.installLocation = installLocation;
           this.signingDetails = signingDetails;
           this.verifiers = verifiers.toArray(new VerifierInfo[verifiers.size()]);
           this.coreApp = coreApp;
           this.debuggable = debuggable;
           this.multiArch = multiArch;
           this.use32bitAbi = use32bitAbi;
           this.extractNativeLibs = extractNativeLibs;
           this.isolatedSplits = isolatedSplits;
       }
</code></pre>

<p><strong>备注</strong>：PackageLite和ApkLite代表不同的含义，前者是包，后者是指apk，一个包中可能包含多个apk</p>
<h4 id="3-1-7-SplitNameComparator">3.1.7  SplitNameComparator</h4><p>表示类比较器，在拆包中排序用到</p>
<pre><code>
/**
    * Used to sort a set of APKs based on their split names, always placing the
    * base APK (with {@code null} split name) first.
    */
   private static class SplitNameComparator implements Comparator&lt;String&gt; {
       @Override
       public int compare(String lhs, String rhs) {
           if (lhs == null) {
               return -1;
           } else if (rhs == null) {
               return 1;
           } else {
               return lhs.compareTo(rhs);
           }
       }
   }
</code></pre>
<h4 id="3-1-8-Package"><a href="#3-1-8-Package" class="headerlink" title="3.1.8  Package"></a>3.1.8  Package</h4><p>表示从磁盘上的apk文件解析出来的完整包，一个包由一个基础的apk和多个拆分的apk构成。</p>
<pre><code>
 /**
     * Representation of a full package parsed from APK files on disk. A package
     * consists of a single base APK, and zero or more split APKs.
     */
    public final static class Package implements Parcelable {
        //表示包名
        @UnsupportedAppUsage
        public String packageName;
        //表示manifest中声明的包名
        // The package name declared in the manifest as the package can be
        // renamed, for example static shared libs use synthetic package names.
        public String manifestPackageName;
        //表示拆包的包名
        /** Names of any split APKs, ordered by parsed splitName */
        public String[] splitNames;

        // TODO: work towards making these paths invariant
        //对应一个volume的uid
        public String volumeUuid;

        /**
         * Path where this package was found on disk. For monolithic packages
         * this is path to single base APK file; for cluster packages this is
         * path to the cluster directory.
         */
         //表示代码路径
        public String codePath;

        /** Path of base APK */
        //表示base APK路径
        public String baseCodePath;
        //表示拆分APK路径
        /** Paths of any split APKs, ordered by parsed splitName */ 
        public String[] splitCodePaths;
        //表示base APK调整版本号
        /** Revision code of base APK */
        public int baseRevisionCode;
         //表示拆分APK调整版本号
        /** Revision codes of any split APKs, ordered by parsed splitName */
        public int[] splitRevisionCodes;
        
         //表示拆分APK的标注数组
        /** Flags of any split APKs; ordered by parsed splitName */
        public int[] splitFlags;

        /**
         * Private flags of any split APKs; ordered by parsed splitName.
         *
         * {@hide}
         */
         //表示拆分APK的私有标注数组
        public int[] splitPrivateFlags;
        //表示是否支持硬件加速
        public boolean baseHardwareAccelerated;

        //对应application对象，对应AndroidManifest里面的<Application>
        // For now we only support one application per package.
        @UnsupportedAppUsage
        public ApplicationInfo applicationInfo = new ApplicationInfo();
        
        //apk安装包中，对应AndroidManifest里面的<Permission>
        @UnsupportedAppUsage
        public final ArrayList<Permission> permissions = new ArrayList<Permission>(0);
        //apk安装包中，对应AndroidManifest里面的<PermissionGroup>
        @UnsupportedAppUsage
        public final ArrayList<PermissionGroup> permissionGroups = new ArrayList<PermissionGroup>(0);
        //apk安装包中，对应AndroidManifest里面的<Activity>
        //不是通常说的Activity，而是PackageParse的内部类Activity
        @UnsupportedAppUsage
        public final ArrayList<Activity> activities = new ArrayList<Activity>(0);
        
        //apk安装包中，对应AndroidManifest里面的<Receiver>
        //不是通常说的Activity，而是PackageParse的内部类Activity
        @UnsupportedAppUsage
        public final ArrayList<Activity> receivers = new ArrayList<Activity>(0);
        
        //apk安装包中，对应AndroidManifest里面的<Provider>
        //不是通常说的Provider，而是PackageParse的内部类Provider
        @UnsupportedAppUsage
        public final ArrayList<Provider> providers = new ArrayList<Provider>(0);
        
        //apk安装包中，对应AndroidManifest里面的<Service>
        //不是通常说的Service，而是PackageParse的内部类Service
        @UnsupportedAppUsage
        public final ArrayList<Service> services = new ArrayList<Service>(0);
        
        //apk安装包中，对应AndroidManifest里面的<Instrumentation>
        //不是通常说的Instrumentation，而是内部类Instrumentation
        @UnsupportedAppUsage
        public final ArrayList<Instrumentation> instrumentation = new ArrayList<Instrumentation>(0);
        //apk安装包中请求的权限
        @UnsupportedAppUsage
        public final ArrayList<String> requestedPermissions = new ArrayList<String>();
        
        //apk安装包中保内广播的action
        @UnsupportedAppUsage
        public ArrayList<String> protectedBroadcasts;
        //父包
        public Package parentPackage;
        //子包
        public ArrayList<Package> childPackages;
        //共享库名称
        public String staticSharedLibName = null;
        //共享库版本
        public long staticSharedLibVersion = 0;
        //依赖库名称
        public ArrayList<String> libraryNames = null;
        @UnsupportedAppUsage
        //使用库名称
        public ArrayList<String> usesLibraries = null;
        //使用静态库名称
        public ArrayList<String> usesStaticLibraries = null;
        //使用静态库版本
        public long[] usesStaticLibrariesVersions = null;
        public String[][] usesStaticLibrariesCertDigests = null;
         //使用其它的库
        @UnsupportedAppUsage
        public ArrayList<String> usesOptionalLibraries = null;
         //使用静态库文件
        @UnsupportedAppUsage
        public String[] usesLibraryFiles = null;
         //使用共享库信息
        public ArrayList<SharedLibraryInfo> usesLibraryInfos = null;
         //安装包中某个Activity信息的集合 在AndroidManifest里面对应<preferred>标签
        public ArrayList<ActivityIntentInfo> preferredActivityFilters = null;
         //安装包中 AndroidManifest中对应original-package的集合
        public ArrayList<String> mOriginalPackages = null;
         //真实包名，通常和mOriginalPackages一起使用
        public String mRealPackage = null;
        //APK安装包 AndroidManifest中对应adopt-permissions的集合
        public ArrayList<String> mAdoptPermissions = null;
       //独立的存储应用程序元数据，避免多个不需要的引用
        // We store the application meta-data independently to avoid multiple unwanted references
        @UnsupportedAppUsage
        public Bundle mAppMetaData = null;
        //版本号
        // The version code declared for this package.
        @UnsupportedAppUsage
        public int mVersionCode;
        //主版本号
        // The major version code declared for this package.
        public int mVersionCodeMajor;
        //长版本号   
        // Return long containing mVersionCode and mVersionCodeMajor.
        public long getLongVersionCode() {
            return PackageInfo.composeLongVersionCode(mVersionCodeMajor, mVersionCode);
        }
        //版本名
        // The version name declared for this package.
        @UnsupportedAppUsage
        public String mVersionName;
        //共享用户Id
        // The shared user id that this package wants to use.
        @UnsupportedAppUsage
        public String mSharedUserId;
        //共享用户标签
        // The shared user label that this package wants to use.
        @UnsupportedAppUsage
        public int mSharedUserLabel;
        //签名
        // Signatures that were read from the package.
        @UnsupportedAppUsage
        @NonNull public SigningDetails mSigningDetails = SigningDetails.UNKNOWN;
        //dexopt的位置，以便pkms跟踪执行dexopt的位置
        // For use by package manager service for quick lookup of
        // preferred up order.
        @UnsupportedAppUsage
        public int mPreferredOrder = 0;
        //最后一次使用package的时间
        // For use by package manager to keep track of when a package was last used.
        public long[] mLastPackageUsageTimeInMills =
                new long[PackageManager.NOTIFY_PACKAGE_USE_REASONS_COUNT];

        // // User set enabled state.
        // public int mSetEnabled = PackageManager.COMPONENT_ENABLED_STATE_DEFAULT;
        //
        // // Whether the package has been stopped.
        // public boolean mSetStopped = false;
        
        //附加数据
        // Additional data supplied by callers.
        @UnsupportedAppUsage
        public Object mExtras;
         
        //硬件配置信息，对应AndroidManifest里面的<users-configuration>标签
        // Applications hardware preferences
        @UnsupportedAppUsage
        public ArrayList<ConfigurationInfo> configPreferences = null;
        //特性组信息，对应AndroidManifest里面的<uses-feature>标签 
        // Applications requested features
        @UnsupportedAppUsage
        public ArrayList<FeatureInfo> reqFeatures = null;
         //特性组信息，对应AndroidManifest里面的<feature-group>标签 
        // Applications requested feature groups
        public ArrayList<FeatureGroupInfo> featureGroups = null;
        
        //安装的属性
        @UnsupportedAppUsage
        public int installLocation;

        //是否是核心
        public boolean coreApp;
        
        //是否是全局必要，所有用户都需要的应用程序，无法为用户卸载
        /* An app that's required for all users and cannot be uninstalled for a user */
        public boolean mRequiredForAllUsers;
        
        //受限账户的验证类型
        /* The restricted account authenticator type that is used by this application */
        public String mRestrictedAccountType;
        //账户的类型
        /* The required account type without which this application will not function */
        public String mRequiredAccountType;
        
        //对应AndroidManifest里面的<overlay>标签 
        public String mOverlayTarget;
        //overlay类别
        public String mOverlayCategory;
        //overlay优先等级
        public int mOverlayPriority;
        //是否是静态overlay
        public boolean mOverlayIsStatic;
        //编译sdk版本
        public int mCompileSdkVersion;
         //编译sdk版本名称
        public String mCompileSdkVersionCodename;
        
        //下面用来给KeySetManagerService的数据
        /**
         * Data used to feed the KeySetManagerService
         */
        @UnsupportedAppUsage
        public ArraySet<String> mUpgradeKeySets; //升级
        @UnsupportedAppUsage
        public ArrayMap<String, ArraySet<PublicKey>> mKeySetMapping; //公钥

        //有abi的话覆盖
        /**
         * The install time abi override for this package, if any.
         *
         * TODO: This seems like a horrible place to put the abiOverride because
         * this isn't something the packageParser parsers. However, this fits in with
         * the rest of the PackageManager where package scanning randomly pushes
         * and prods fields out of {@code this.applicationInfo}.
         */
        public String cpuAbiOverride;
        //是否用32位的abi
        /**
         * The install time abi override to choose 32bit abi's when multiple abi's
         * are present. This is only meaningfull for multiarch applications.
         * The use32bitAbi attribute is ignored if cpuAbiOverride is also set.
         */
        public boolean use32bitAbi;
        //限制升级hash
        public byte[] restrictUpdateHash;
        //是否对InstantApps可见
        /** Set if the app or any of its components are visible to instant applications. */
        public boolean visibleToInstantApps;
        //是否是备份
        /** Whether or not the package is a stub and must be replaced by the full version. */
        public boolean isStub;

        @UnsupportedAppUsage
        public Package(String packageName) {
            this.packageName = packageName;
            this.manifestPackageName = packageName;
            applicationInfo.packageName = packageName;
            applicationInfo.uid = -1;
        }
        ...
}

</code></pre>
<h4 id="3-1-9-Component"><a href="#3-1-9-Component" class="headerlink" title="3.1.9  Component"></a>3.1.9  Component</h4>
<pre><code>
public static abstract class IntentInfo extends IntentFilter {
        //是否有默认
        @UnsupportedAppUsage
        public boolean hasDefault;
        //标签的资源id
        @UnsupportedAppUsage
        public int labelRes;
         //本地化的标签
        @UnsupportedAppUsage
        public CharSequence nonLocalizedLabel;
         //icon的资源id
        @UnsupportedAppUsage
        public int icon;
         //logo的资源id
        @UnsupportedAppUsage
        public int logo;
         //banner的资源id
        @UnsupportedAppUsage
        public int banner;
         //preferred的资源id
        public int preferred;
        ...
   }
 
 public static abstract class Component<II extends IntentInfo> {
        //该组件所包含的IntentFilter
        @UnsupportedAppUsage
        public final ArrayList<II> intents; 
        //该组件的类名
        @UnsupportedAppUsage
        public final String className;
         //该组件的元数据
        @UnsupportedAppUsage
        public Bundle metaData;
         //该组件的类名
        @UnsupportedAppUsage
        //包含该组件的包名
        public Package owner;
        /** The order of this component in relation to its peers */
        public int order;
        //该组件名
        ComponentName componentName;
         //组件短名
        String componentShortName;
        ...
}

</code></pre>
<h4 id="3-1-10-Permission"><a href="#3-1-10-Permission" class="headerlink" title="3.1.10  Permission"></a>3.1.10  Permission</h4><p>继承于Component，对应AndroidManifest里面的<permission>标签</permission></p>
<pre><code>
public final static class Permission extends Component<IntentInfo> implements Parcelable {
        //权限信息
        @UnsupportedAppUsage
        public final PermissionInfo info;
        //是否权限树
        @UnsupportedAppUsage
        public boolean tree;
        //对应的权限组
        @UnsupportedAppUsage
        public PermissionGroup group;

        public Permission(Package _owner) {
            super(_owner);
            info = new PermissionInfo();
        }
        ...
}

</code></pre>
<p>继承于Component，对应AndroidManifest里面的<activity>标签</activity></p>
<pre><code>
public final static class Activity extends Component<ActivityIntentInfo> implements Parcelable {
        @UnsupportedAppUsage
        public final ActivityInfo info;
        private boolean mHasMaxAspectRatio;

        private boolean hasMaxAspectRatio() {
            return mHasMaxAspectRatio;
        }

        public Activity(final ParseComponentArgs args, final ActivityInfo _info) {
            super(args, _info);
            info = _info;
            info.applicationInfo = args.owner.applicationInfo;
        }

        public void setPackageName(String packageName) {
            super.setPackageName(packageName);
            info.packageName = packageName;
        }
        ...
}

</code></pre>
<p>一些其他的基本类似类在这里就不再详细的介绍，下面看下类里面的一些方法：</p>
<h3 id="3-2-内部方法"><a href="#3-2-内部方法" class="headerlink" title="3.2 内部方法"></a>3.2 内部方法</h3><p>里面的方法基本上是和解析相关的方法，这里以parseActivity为例说明，其他的解析大同小异。</p>
<h4 id="3-2-1-parsePackage"><a href="#3-2-1-parsePackage" class="headerlink" title="3.2.1 parsePackage"></a>3.2.1 parsePackage</h4><p>这个类是解析package最开始的方法，其他的解析方法都是从这个入口进入的，这里分为两种解析，一种是single APK ，另一种是cluster APKs。</p>
<pre><code>
/**
    * Parse the package at the given location. Automatically detects if the
    * package is a monolithic style (single APK file) or cluster style
    * (directory of APKs).
    * <p>
    * This performs sanity checking on cluster style packages, such as
    * requiring identical package name and version codes, a single base APK,
    * and unique split names.
    * <p>
    * Note that this <em>does not</em> perform signature verification; that
    * must be done separately in {@link #collectCertificates(Package, int)}.
    *
    * If {@code useCaches} is true, the package parser might return a cached
    * result from a previous parse of the same {@code packageFile} with the same
    * {@code flags}. Note that this method does not check whether {@code packageFile}
    * has changed since the last parse, it's up to callers to do so.
    *
    * @see #parsePackageLite(File, int)
    */
   @UnsupportedAppUsage
   public Package parsePackage(File packageFile, int flags, boolean useCaches)
           throws PackageParserException {
       Package parsed = useCaches ? getCachedResult(packageFile, flags) : null;
       if (parsed != null) {
           return parsed;
       }

       long parseTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
       //Cluster app
       if (packageFile.isDirectory()) {
           parsed = parseClusterPackage(packageFile, flags);
       } else {
           parsed = parseMonolithicPackage(packageFile, flags);
       }

       long cacheTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
       cacheResult(packageFile, flags, parsed);
       if (LOG_PARSE_TIMINGS) {
           parseTime = cacheTime - parseTime;
           cacheTime = SystemClock.uptimeMillis() - cacheTime;
           if (parseTime + cacheTime > LOG_PARSE_TIMINGS_THRESHOLD_MS) {
               Slog.i(TAG, "Parse times for '" + packageFile + "': parse=" + parseTime
                       + "ms, update_cache=" + cacheTime + " ms");
           }
       }
       return parsed;
   }

</code></pre>
<h4 id="3-2-2-parseActivity"><a href="#3-2-2-parseActivity" class="headerlink" title="3.2.2 parseActivity"></a>3.2.2 parseActivity</h4><p>这个方法主要是解析AndroidManifest中activity标签的内容，并将其保存到PackageParser.Activity对象中。</p>
<pre><code>
private Activity parseActivity(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError, CachedComponentArgs cachedArgs,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {
        //获取资源数组
        TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestActivity);
        //初始化解析Activity参数
        if (cachedArgs.mActivityArgs == null) {
            cachedArgs.mActivityArgs = new ParseComponentArgs(owner, outError,
                    R.styleable.AndroidManifestActivity_name,
                    R.styleable.AndroidManifestActivity_label,
                    R.styleable.AndroidManifestActivity_icon,
                    R.styleable.AndroidManifestActivity_roundIcon,
                    R.styleable.AndroidManifestActivity_logo,
                    R.styleable.AndroidManifestActivity_banner,
                    mSeparateProcesses,
                    R.styleable.AndroidManifestActivity_process,
                    R.styleable.AndroidManifestActivity_description,
                    R.styleable.AndroidManifestActivity_enabled);
        }
        //判断是receiver还是activity
        cachedArgs.mActivityArgs.tag = receiver ? "<receiver>" : "<activity>";
        cachedArgs.mActivityArgs.sa = sa;
        cachedArgs.mActivityArgs.flags = flags;
        //创建Activity类，是ParserPackage内部类
        Activity a = new Activity(cachedArgs.mActivityArgs, new ActivityInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }
        //是否在AndroidManifest里面设置exported属性
        boolean setExported = sa.hasValue(R.styleable.AndroidManifestActivity_exported);
        if (setExported) {
            a.info.exported = sa.getBoolean(R.styleable.AndroidManifestActivity_exported, false);
        }
         //获取AndroidManifest里面对应的theme的值
        a.info.theme = sa.getResourceId(R.styleable.AndroidManifestActivity_theme, 0);
        //获取AndroidManifest里面对应的uiOptions的值
        a.info.uiOptions = sa.getInt(R.styleable.AndroidManifestActivity_uiOptions,
                a.info.applicationInfo.uiOptions);
         //获取AndroidManifest里面对应的parentActivityName的值
        String parentName = sa.getNonConfigurationString(
                R.styleable.AndroidManifestActivity_parentActivityName,
                Configuration.NATIVE_CONFIG_VERSION);
          //如果设置了
        if (parentName != null) {
           //构建parent的标签
            String parentClassName = buildClassName(a.info.packageName, parentName, outError);
            if (outError[0] == null) {
                a.info.parentActivityName = parentClassName;
            } else {
                Log.e(TAG, "Activity " + a.info.name + " specified invalid parentActivityName " +
                        parentName);
                outError[0] = null;
            }
        }
         //获取权限permission
        String str;
        str = sa.getNonConfigurationString(R.styleable.AndroidManifestActivity_permission, 0);
        if (str == null) {
            a.info.permission = owner.applicationInfo.permission;
        } else {
            a.info.permission = str.length() > 0 ? str.toString().intern() : null;
        }

        //获取AndroidManifest里面对应的taskAffinity的值
        str = sa.getNonConfigurationString(
                R.styleable.AndroidManifestActivity_taskAffinity,
                Configuration.NATIVE_CONFIG_VERSION);
        a.info.taskAffinity = buildTaskAffinityName(owner.applicationInfo.packageName,
                owner.applicationInfo.taskAffinity, str, outError);
        
         //获取AndroidManifest里面对应的splitName的值
        a.info.splitName =
                sa.getNonConfigurationString(R.styleable.AndroidManifestActivity_splitName, 0);

         //是否在AndroidManifest里面的Activity配置了multiprocesss属性
        a.info.flags = 0;
        if (sa.getBoolean(
                R.styleable.AndroidManifestActivity_multiprocess, false)) {
            a.info.flags |= ActivityInfo.FLAG_MULTIPROCESS;
        }
         //是否在AndroidManifest里面的Activity配置了finishOnTaskLaunch属性
         //用来标示用户再次启动任务时，是否应该关闭现有的Activity
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_finishOnTaskLaunch, false)) {
            a.info.flags |= ActivityInfo.FLAG_FINISH_ON_TASK_LAUNCH;
        }
        //是否在AndroidManifest里面的Activity配置了clearTaskOnLaunch属性
        //用来标示当前应用从主屏幕重新启动时是否都从中移除根activity之外的其他activity
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_clearTaskOnLaunch, false)) {
            a.info.flags |= ActivityInfo.FLAG_CLEAR_TASK_ON_LAUNCH;
        }
        //是否在AndroidManifest里面的Activity配置了noHistory属性
        //用来标示当前用户离开activity时，是否从activity堆栈中移除
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_noHistory, false)) {
            a.info.flags |= ActivityInfo.FLAG_NO_HISTORY;
        }
        //是否在AndroidManifest里面的Activity配置了alwaysRetainTaskState属性
        //用来标示当是否保持activity所在的任务状态
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_alwaysRetainTaskState, false)) {
            a.info.flags |= ActivityInfo.FLAG_ALWAYS_RETAIN_TASK_STATE;
        }
        //是否在AndroidManifest里面的Activity配置了stateNotNeeded属性
        //用来标示是否不保存activity状态的情况下将其终止并成功重启
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_stateNotNeeded, false)) {
            a.info.flags |= ActivityInfo.FLAG_STATE_NOT_NEEDED;
        }
        //是否在AndroidManifest里面的Activity配置了excludeFromRecents属性
        //用来标示activity启动任务在最近使用的应用列表之外
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_excludeFromRecents, false)) {
            a.info.flags |= ActivityInfo.FLAG_EXCLUDE_FROM_RECENTS;
        }
        //是否在AndroidManifest里面的Activity配置了allowTaskReparenting属性
        //用来标示这个应用是否在reset task时，关联对应的taskAffinity
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_allowTaskReparenting,
                (owner.applicationInfo.flags&ApplicationInfo.FLAG_ALLOW_TASK_REPARENTING) != 0)) {
            a.info.flags |= ActivityInfo.FLAG_ALLOW_TASK_REPARENTING;
        }
        //是否在AndroidManifest里面的Activity配置了finishOnCloseSystemDialogs属性
        //用来标示在关闭系统窗口时，是否销毁activity
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_finishOnCloseSystemDialogs, false)) {
            a.info.flags |= ActivityInfo.FLAG_FINISH_ON_CLOSE_SYSTEM_DIALOGS;
        }
        //是否在AndroidManifest里面的Activity配置了showForAllUsers属性
        //用来指定该activity是否显示在解锁界面
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_showOnLockScreen, false)
                || sa.getBoolean(R.styleable.AndroidManifestActivity_showForAllUsers, false)) {
            a.info.flags |= ActivityInfo.FLAG_SHOW_FOR_ALL_USERS;
        }
        //是否在AndroidManifest里面的Activity配置了immersive属性
        //是否设置沉浸式显示
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_immersive, false)) {
            a.info.flags |= ActivityInfo.FLAG_IMMERSIVE;
        }
         //是否在AndroidManifest里面的Activity配置了systemUserOnly属性
           //是否设置为系统组件
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_systemUserOnly, false)) {
            a.info.flags |= ActivityInfo.FLAG_SYSTEM_USER_ONLY;
        }

        //不是receiver标签
        if (!receiver) {
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_hardwareAccelerated,
                    hardwareAccelerated)) {
                a.info.flags |= ActivityInfo.FLAG_HARDWARE_ACCELERATED;
            }
            //设置对应的启动模式，对应的launchMode
            a.info.launchMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_launchMode, ActivityInfo.LAUNCH_MULTIPLE);
            //设置对应的启动模式，对应的documentLaunchMode
            //一下也是获取一序列的属性，在此不再注解
            a.info.documentLaunchMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_documentLaunchMode,
                    ActivityInfo.DOCUMENT_LAUNCH_NONE);
            a.info.maxRecents = sa.getInt(
                    R.styleable.AndroidManifestActivity_maxRecents,
                    ActivityManager.getDefaultAppRecentsLimitStatic());
            a.info.configChanges = getActivityConfigChanges(
                    sa.getInt(R.styleable.AndroidManifestActivity_configChanges, 0),
                    sa.getInt(R.styleable.AndroidManifestActivity_recreateOnConfigChanges, 0));
            a.info.softInputMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_windowSoftInputMode, 0);

            a.info.persistableMode = sa.getInteger(
                    R.styleable.AndroidManifestActivity_persistableMode,
                    ActivityInfo.PERSIST_ROOT_ONLY);

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_allowEmbedded, false)) {
                a.info.flags |= ActivityInfo.FLAG_ALLOW_EMBEDDED;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_autoRemoveFromRecents, false)) {
                a.info.flags |= ActivityInfo.FLAG_AUTO_REMOVE_FROM_RECENTS;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_relinquishTaskIdentity, false)) {
                a.info.flags |= ActivityInfo.FLAG_RELINQUISH_TASK_IDENTITY;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_resumeWhilePausing, false)) {
                a.info.flags |= ActivityInfo.FLAG_RESUME_WHILE_PAUSING;
            }

            a.info.screenOrientation = sa.getInt(
                    R.styleable.AndroidManifestActivity_screenOrientation,
                    SCREEN_ORIENTATION_UNSPECIFIED);

            setActivityResizeMode(a.info, sa, owner);

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_supportsPictureInPicture,
                    false)) {
                a.info.flags |= FLAG_SUPPORTS_PICTURE_IN_PICTURE;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_alwaysFocusable, false)) {
                a.info.flags |= FLAG_ALWAYS_FOCUSABLE;
            }

            if (sa.hasValue(R.styleable.AndroidManifestActivity_maxAspectRatio)
                    && sa.getType(R.styleable.AndroidManifestActivity_maxAspectRatio)
                    == TypedValue.TYPE_FLOAT) {
                a.setMaxAspectRatio(sa.getFloat(R.styleable.AndroidManifestActivity_maxAspectRatio,
                        0 /*default*/));
            }

            a.info.lockTaskLaunchMode =
                    sa.getInt(R.styleable.AndroidManifestActivity_lockTaskMode, 0);

            a.info.encryptionAware = a.info.directBootAware = sa.getBoolean(
                    R.styleable.AndroidManifestActivity_directBootAware,
                    false);

            a.info.requestedVrComponent =
                sa.getString(R.styleable.AndroidManifestActivity_enableVrMode);

            a.info.rotationAnimation =
                sa.getInt(R.styleable.AndroidManifestActivity_rotationAnimation, ROTATION_ANIMATION_UNSPECIFIED);

            a.info.colorMode = sa.getInt(R.styleable.AndroidManifestActivity_colorMode,
                    ActivityInfo.COLOR_MODE_DEFAULT);

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_showWhenLocked, false)) {
                a.info.flags |= ActivityInfo.FLAG_SHOW_WHEN_LOCKED;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_turnScreenOn, false)) {
                a.info.flags |= ActivityInfo.FLAG_TURN_SCREEN_ON;
            }

        } else {
           //是receiver
            a.info.launchMode = ActivityInfo.LAUNCH_MULTIPLE;
            a.info.configChanges = 0;

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_singleUser, false)) {
                a.info.flags |= ActivityInfo.FLAG_SINGLE_USER;
            }

            a.info.encryptionAware = a.info.directBootAware = sa.getBoolean(
                    R.styleable.AndroidManifestActivity_directBootAware,
                    false);
        }

        if (a.info.directBootAware) {
            owner.applicationInfo.privateFlags |=
                    ApplicationInfo.PRIVATE_FLAG_PARTIALLY_DIRECT_BOOT_AWARE;
        }

        // can't make this final; we may set it later via meta-data
        boolean visibleToEphemeral =
                sa.getBoolean(R.styleable.AndroidManifestActivity_visibleToInstantApps, false);
        if (visibleToEphemeral) {
            a.info.flags |= ActivityInfo.FLAG_VISIBLE_TO_INSTANT_APP;
            owner.visibleToInstantApps = true;
        }

        sa.recycle();

        if (receiver && (owner.applicationInfo.privateFlags
                &ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            // A heavy-weight application can not have receives in its main process
            // We can do direct compare because we intern all strings.
            if (a.info.processName == owner.packageName) {
                outError[0] = "Heavy-weight applications can not have receivers in main process";
            }
        }

        if (outError[0] != null) {
            return null;
        }

        int outerDepth = parser.getDepth();
        int type;
        //开始解析<activity>内部标签
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            //intent-filter标签
            if (parser.getName().equals("intent-filter")) {
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, true /*allowGlobs*/, true /*allowAutoVerify*/,
                        intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in intent filter at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {
                    a.order = Math.max(intent.getOrder(), a.order);
                    a.intents.add(intent);
                }
                // adjust activity flags when we implicitly expose it via a browsable filter
                final int visibility = visibleToEphemeral
                        ? IntentFilter.VISIBILITY_EXPLICIT
                        : !receiver && isImplicitlyExposedIntent(intent)
                                ? IntentFilter.VISIBILITY_IMPLICIT
                                : IntentFilter.VISIBILITY_NONE;
                intent.setVisibilityToInstantApp(visibility);
                if (intent.isVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_VISIBLE_TO_INSTANT_APP;
                }
                if (intent.isImplicitlyVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_IMPLICITLY_VISIBLE_TO_INSTANT_APP;
                }
                if (LOG_UNSAFE_BROADCASTS && receiver
                        && (owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.O)) {
                    for (int i = 0; i < intent.countActions(); i++) {
                        final String action = intent.getAction(i);
                        if (action == null || !action.startsWith("android.")) continue;
                        if (!SAFE_BROADCASTS.contains(action)) {
                            Slog.w(TAG, "Broadcast " + action + " may never be delivered to "
                                    + owner.packageName + " as requested at: "
                                    + parser.getPositionDescription());
                        }
                    }
                }
            } else if (!receiver && parser.getName().equals("preferred")) {
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, false /*allowGlobs*/, false /*allowAutoVerify*/,
                        intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in preferred at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {
                    if (owner.preferredActivityFilters == null) {
                        owner.preferredActivityFilters = new ArrayList<ActivityIntentInfo>();
                    }
                    owner.preferredActivityFilters.add(intent);
                }
                // adjust activity flags when we implicitly expose it via a browsable filter
                final int visibility = visibleToEphemeral
                        ? IntentFilter.VISIBILITY_EXPLICIT
                        : !receiver && isImplicitlyExposedIntent(intent)
                                ? IntentFilter.VISIBILITY_IMPLICIT
                                : IntentFilter.VISIBILITY_NONE;
                intent.setVisibilityToInstantApp(visibility);
                if (intent.isVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_VISIBLE_TO_INSTANT_APP;
                }
                if (intent.isImplicitlyVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_IMPLICITLY_VISIBLE_TO_INSTANT_APP;
                }
            } else if (parser.getName().equals("meta-data")) {
                if ((a.metaData = parseMetaData(res, parser, a.metaData,
                        outError)) == null) {
                    return null;
                }
            } else if (!receiver && parser.getName().equals("layout")) {
                parseLayout(res, parser, a);
            } else {
                if (!RIGID_PARSER) {
                    Slog.w(TAG, "Problem in package " + mArchiveSourcePath + ":");
                    if (receiver) {
                        Slog.w(TAG, "Unknown element under <receiver>: " + parser.getName()
                                + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                    } else {
                        Slog.w(TAG, "Unknown element under <activity>: " + parser.getName()
                                + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                } else {
                    if (receiver) {
                        outError[0] = "Bad element under <receiver>: " + parser.getName();
                    } else {
                        outError[0] = "Bad element under <activity>: " + parser.getName();
                    }
                    return null;
                }
            }
        }

        if (!setExported) {
            a.info.exported = a.intents.size() > 0;
        }

        return a;
    }

</code></pre>
<h3 id="3-3-PackageParser总结"><a href="#3-3-PackageParser总结" class="headerlink" title="3.3 PackageParser总结"></a>3.3 PackageParser总结</h3><p><img src="/2019/PKMS相关类分析/PackageParser.png" alt=""></p>
<p>上图画出了PackageParser解析Apk文件，得到的主要的数据结构，实际的内容远多于这些，我们仅保留了四大组件和权限相关的内容。</p>
<p>上面这些类，全部是定义于PackageParser中的内部类，这些内部类主要的作用就是保存AndroidManifest.xml解析出的对应信息。 以PackageParser.Activity为例，注意到该类持有ActivityInfo类，继承自 Component&lt; ActivityIntentInfo&gt;。其中，ActivityInfo用于保存Activity的信息；Component类是一个模板，对应元素类型是ActivityIntentInfo，顶层基类IntentFilter。四大组件中的其它成员，也有类似的继承结构。</p>
<blockquote>
<p>这种设计的原因是：Package除了保存信息外，还需要支持Intent匹配查询。例如，当收到某个Intent后，由于ActivityIntentInfo继承自IntentFilter，因此它能判断自己是否满足Intent的要求。如果满足，则返回对应的ActivityInfo。</p>
</blockquote>
<p>PackageParser整个扫描过程如下：</p>
<ul>
<li><p>PackageParser首先解析出ApkLite，得到每个Apk文件的简化信息；</p>
</li>
<li><p>利用所有的ApkLite以及Xml中其它信息，解析出PackageLite；</p>
</li>
</ul>
<ul>
<li>利用PackageLite中的信息及XML中的其它信息，解析出Package信息；</li>
</ul>
<p>​       Package中基本上涵盖了AndroidManifest中涉及的所有信息。</p>
<p>注意：在上述的解析过程中，PackageParser利用AssetManager存储了Package中资源文件的地址。</p>
<h3 id="附录"><a href="#附录" class="headerlink" title="附录"></a>附录</h3><p>源码路径</p>
<pre><code>
/frameworks/base/services/core/java/com/android/server/pm/Settings.java
/frameworks/base/core/java/com/android/server/SystemConfig.java
/frameworks/base/core/java/android/content/pm/PackageParser.java

</code></pre>

      