---
layout: post
title:  "Logging Shortcut for Android"
---

If you're an Android programmer, you probably end up reading a lot of logging output. And a lot of log files TODO.

Consider copying the following code in a class full of utility methods:

{% highlight java %}
private static String getTag(Activity activity) {
  return activity.getClass().getSimpleName();
}

/**
 * Delegates to {@link Log#v(String, String)} with the activity class name
 * as the tag.
 */
public static void v(Activity activity, String msg) {
  Log.v(getTag(activity), msg);
}
/**
 * Delegates to {@link Log#v(String, String, Throwable))} with the activity class name
 * as the tag.
 */
public static void v(Activity activity, String msg, Throwable e) {
  Log.v(getTag(activity), msg, e);
}

/**
 * Delegates to {@link Log#d(String, String)} with the activity class name
 * as the tag.
 */
public static void d(Activity activity, String msg) {
  Log.d(getTag(activity), msg);
}
/**
 * Delegates to {@link Log#d(String, String, Throwable)} with the activity class name
 * as the tag.
 */
public static void d(Activity activity, String msg, Throwable e) {
  Log.d(getTag(activity), msg, e);
}

/**
 * Delegates to {@link Log#i(String, String)} with the activity class name
 * as the tag.
 */
public static void i(Activity activity, String msg) {
  Log.i(getTag(activity), msg);
}
/**
 * Delegates to {@link Log#i(String, String, Throwable)} with the activity class name
 * as the tag.
 */
public static void i(Activity activity, String msg, Throwable e) {
  Log.i(getTag(activity), msg, e);
}

/**
 * Delegates to {@link Log#w(String, String)} with the activity class name
 * as the tag.
 */
public static void w(Activity activity, String msg) {
  Log.w(getTag(activity), msg);
}
/**
 * Delegates to {@link Log#w(String, String, Throwable)} with the activity class name
 * as the tag.
 */
public static void w(Activity activity, String msg, Throwable e) {
  Log.w(getTag(activity), msg, e);
}

/**
 * Delegates to {@link Log#e(String, String)} with the activity class name
 * as the tag.
 */
public static void e(Activity activity, String msg) {
  Log.e(getTag(activity), msg);
}
/**
 * Delegates to {@link Log#e(String, String, Throwable)} with the activity class name
 * as the tag.
 */
public static void e(Activity activity, String msg, Throwable e) {
  Log.e(getTag(activity), msg, e);
}
{% endhighlight %}

Say I copy this code into `Util.java`. Within any `Activity` instance I can . For example, consider the following code from `com.readyupapp.LoginActivity` that handles a login request:

{% highlight java %}
try {
  String authToken = server.login(username, password);
} catch (AuthenticationException e) {
  Util.i(this, "Login failed for user: " + username, e);

  showLoginFailedAlertDialog();
  return;
}
{% endhighlight %}

This calls method `Util.i(Activity, String, Throwable)`, passing the `LoginActivity` instance as the first `Activity` parameter. In the log file, the tag is automatically displayed as `LoginActivity`, which is the string returned by method `getSimpleName()`.

Method `getSimpleName()` returns the empty string for an anonymous inner class, but it's highly unlikely that you will have such an anonymous inner class that extends `Activity`. And of course, the preceding methods can only be called with instances that extend `Activity`. You could change each method to accept an `Object` instance instead of an `Activity` instance, and then redefine `getTag` to guard against returning the empty string as a tag, like so:

{% highlight java %}
private static String getTag(Object obj) {
  String tag = obj.getClass().getSimpleName();
  if ((tag == null) || tag.isEmpty()) {
    // Return the fully qualified, if sometimes ugly, name.
    tag = obj.getClass().getName();
  }
  return tag;
}
{% endhighlight %}

This makes it too easy, however, to call the method with an anonymous inner class. For example, consider the following code, also from class `LoginActivity`:

{% highlight java %}
new AlertDialog.Builder(this)
    .setTitle(R.string.delete_contact_title)
    .setMessage(R.string.delete_contact_message)
    .setPositiveButton(
        R.string.btn_delete_contact, new DialogInterface.OnClickListener() {
      @Override
      public void onClick(DialogInterface dialog, int id) {
        Util.i(this, "Deleting contact through the alert");
        deleteContact();
      }
    })
    .setNegativeButton(R.string.btn_cancel, Util.EMPTY_DIALOG_LISTENER)
    .create()
    .show();
{% endhighlight %}

Instead of passing the `LoginActivity` instance as the first parameter to `Util.i(Object, String)`, this actually passes the anonymous `DialogInterface.OnClickListener` instance. Its corresponding tag, returned by method `getName()`, is not as readable as the tag `"LoginActivity"`. Moreover, the logged text already specifies that we are deleting the contact through the alert; it is redundant to also encode this information in the tag.

To pass the `Activity` instance instead, we must qualify the `this` keyword with the class name `LoginActivity` when calling `Util.i(Object, String)`, like so:

{% highlight java %}
@Override
public void onClick(DialogInterface dialog, int id) {
  Util.i(LoginActivity.this, "Deleting contact through the alert");
  deleteContact();
}
{% endhighlight %}

If we did not change each method to accept an `Object` instance instead of an `Activity` instance, this mistake would have been caught at compile time, as `DialogInterface.OnClickListener` does not extend `Activity`.

