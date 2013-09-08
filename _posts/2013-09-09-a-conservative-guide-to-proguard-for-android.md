---
layout: post
title:  "A Conservative Guide to ProGuard for Android"
tags: ["guide", "proguard", "android"]
---

Android's [documentation for ProGuard](http://developer.android.com/tools/help/proguard.html) describes it like so:

> "The ProGuard tool shrinks, optimizes, and obfuscates your code by removing unused code and renaming classes, fields, and methods with semantically obscure names. The result is a smaller sized `.apk` file that is more difficult to reverse engineer.... Having ProGuard run is completely optional, but highly recommended."

But this documentation for ProGuard is minimal in its recommendations, and [ProGuard advice on Stack Overflow](http://stackoverflow.com/questions/tagged/proguard) is inconsistent. Some developers find that ProGuard can introduce bugs when delivering push notifications, making in-app purchases, and using third-party libraries. This post is a *conservative guide* to ProGuard that minimizes the chance of these bugs by stressing safety over aggressive optimizations. When developing [Burner for Android](https://play.google.com/store/apps/details?id=com.adhoclabs.burner), this approach still led to ProGuard reducing the size of its `.apk` file by almost 25%, from ~1.7 MB to ~1.3 MB. 

## Basic configuration

First, open your project's `project.properties` file and find the line specifying `proguard.config`. Append to it a colon as a delimiter, followed by `proguard-appname.txt`, substituting your own application name for `appname`. In its entirety the line should look like:

`proguard.config=${sdk.dir}/tools/proguard/proguard-android.txt:proguard-appname.txt`

So for Burner, it looks like:

`proguard.config=${sdk.dir}/tools/proguard/proguard-android.txt:proguard-burner.txt`

If no such line for `proguard.config` exists in your `project.properties` file, add one.

If you navigate to the directory `{sdk.dir}/tools/proguard/`, you'll find that in addition to `proguard-android.txt` there is another file named `proguard-android-optimize.txt`. By using the former, your code will be shrunk and obfuscated. By using the latter, your code will also be optimized. But as Donald Knuth famously said, premature optimization is the root of all evil. The preamble to `proguard-android-optimize.txt` warns us so:

> "Adding optimization introduces certain risks, since for example not all optimizations performed by ProGuard works on all versions of Dalvik. The following flags turn off various optimizations known to have issues, but the list may not be complete or up to date."

We're erring on the side of safety and referring to `proguard-android.txt` instead of `proguard-android-optimize.txt`. If your application is running slowly, it may be best to explore other [space-time tradeoffs](http://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff) in your code before switching to `proguard-android-optimize.txt`. If using `proguard-android-optimize.txt` introduces a bug on an esoteric or new version of Dalvik, that bug may be impossible or costly to fix.

Next, create that file `proguard-appname.txt` in your project. It should be a sibling of the `project.properties` file. Again, for Burner, this filename is `proguard-burner.txt`.

From the collective wisdom of Stack Overflow, I always begin this file with the following entries:

```text
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.preference.Preference
-keep public class com.android.vending.billing.IInAppBillingService
-keep public class * extends android.view.View {
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
    public <init>(android.content.Context, android.util.AttributeSet, int);
}
-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet);
}
-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet, int);
}
-keepclassmembers class * extends android.content.Context {
    public void *(android.view.View);
    public void *(android.view.MenuItem);
}
```

This excludes shrinking and obfuscating parts of classes that extend select classes from the Android API. The line including `BroadcastReceiver` is especially important if your application reacts to notifications, while the line including `IInAppBillingService` is especially important if your application offers in-app purchases. Again, some of these exclusionary rules could probably be removed, but we're erring on the side of safety.

## Third-party libraries

Many problems with ProGuard stem from the inclusion of third-party libraries in your project. For example, the following line of code from the [greenDAO ORM](http://greendao-orm.com/) throws a [`NoSuchFieldException`](http://developer.android.com/reference/java/lang/NoSuchFieldException.html) because ProGuard obfuscates the field name `TABLENAME`:

```java
this.tablename = (String) daoClass.getField("TABLENAME").get(null);
```

The first step in avoiding such errors is to consult the documentation of each third-party library and follow any instructions for deployment with ProGuard. For example, the [Urban Airship documentation](https://support.urbanairship.com/customer/portal/articles/241918-deploying-your-android-app-with-proguard) instructs adding the following lines to your configuration:

```text
# Suppress warnings if you are NOT using IAP:
-dontwarn com.urbanairship.iap.**

# Required if you are using Autopilot
-keep public class * extends com.urbanairship.Autopilot

# Required if you ARE using IAP:
#-keep class com.android.vending.billing.**

# Required if you are using the airshipconfig.properties file
-keepclasseswithmembers public class * extends com.urbanairship.Options {
    public *;
}
```

If the documentation mentions nothing about ProGuard, but the library is open source, then there is no point to obfuscating its code from a security standpoint anyway. Find the source code for the library, or temporarily rename the JAR file so that it has a `.zip` extension and extract its contents. Then find the ancestor packages for its code. 
For example, the ancestor package for all greenDAO ORM code is `de.greenrobot.dao`, which includes subpackages `async`, `identityscope`, `internal`, `query`, and `test`. With the following two lines in our ProGuard configuration, we exclude shrinking and obfuscating any code in package `de.greenrobot.dao` or a subpackage like `de.greenrobot.dao.async`:

```text
-libraryjars libs

-keep class de.greenrobot.dao.** { *; }
-keep interface de.greenrobot.dao.** { *; }
```

In addition to the JAR file for greenDAO ORM, your `libs` folder may contain JAR files for third-party libraries like [Joda Time](http://joda-time.sourceforge.net/) and the [Android Asynchronous Http Client](http://loopj.com/android-async-http/). Your project may also depend on Android library projects like [ActionBarSherlock](http://actionbarsherlock.com/) and [ViewPagerIndicator](http://viewpagerindicator.com/). Applying the approach above to each of these libraries, `proguard-android.txt` looks like:

```text
-libraryjars libs

# The official support library.
-keep class android.support.v4.app.** { *; }
-keep interface android.support.v4.app.** { *; }

# Library JARs.
-keep class de.greenrobot.dao.** { *; }
-keep interface de.greenrobot.dao.** { *; }
-keep class org.joda.** { *; }
-keep interface org.joda.** { *; }
-keep class com.loopj.android.http.** { *; }
-keep interface com.loopj.android.http.** { *; }

# Library projects.
-keep class com.actionbarsherlock.** { *; }
-keep interface com.actionbarsherlock.** { *; }
-keep class com.viewpagerindicator.** { *; }
-keep interface com.viewpagerindicator.** { *; }
```

This immediately follows the block of entries shown under *Basic configuration*. Change its content according to what libraries your project depends on.

## Output

ProGuard runs when you build your application in release mode. After it has run, you should find a `proguard` directory in your project directory. The [Android's documentation for ProGuard](http://developer.android.com/tools/help/proguard.html) contains complete information on the files it contains, but two are especially important:

* `seeds.txt` contains classes and members excluded from obfuscation. Here you should find those belonging to the library packages that you explicitly excluded; if missing, then they were obfuscated and bugs may appear.
* `mapping.txt` maps between the original and obfuscated class, method, and field names. If you are using a crash-reporting tool like [Crittercism](https://www.crittercism.com/) to automatically collect the stacktraces from crashes, then you will need this file to translate their obfuscated names back to their original names. Many crash-reporting tools will automatically perform this translation given this file; for example, see the [Crittercism documentation](https://app.crittercism.com/developers/docs-android). Because `mapping.txt` is so important to debugging, commit it to Git, add it to Dropbox, or copy it to a location that grants it longevity.

## Further optimization

While this is a conservative guide to ProGuard, you can use it as a starting point for aggressive optimization with fewer exclusionary rules. Refer to its [official documentation](http://proguard.sourceforge.net/) for more information.

