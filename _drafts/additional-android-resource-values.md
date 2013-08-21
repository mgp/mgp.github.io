---
layout: post
title:  "Additional Android Resource Values"
---

Most Android developers associate the `values` resources folder with simple `string` elements in `strings.xml`. But [buried in the Android documentation](http://developer.android.com/guide/topics/resources/more-resources.html) are some additional resource types that make development easier and your app 

## Dimensions

In a file named `layout.xml`, we can define dimension values like so:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <dimen name="margin_large">24dp</dimen>
    <dimen name="margin_small">12dp</dimen>
    <dimen name="button_small_height">36dp</dimen>
    <dimen name="button_large_height">52dp</dimen>
    <dimen name="icon_size">32dp</dimen>
</resources>
{% endhighlight %}

Note that we're using `dp`, or [density-independent pixels](http://developer.android.com/guide/topics/resources/more-resources.html#Dimension), instead of `sp`. It is recommended that you use the latter only when specifying font sizes, and the former in all other cases. Once we've defined these values, we can reference them in layout XML files by using the prefix `@dimen`. For example, the following `ImageView` element in a `RelativeLayout` defines its right margin as `margin_small`:

{% highlight xml %}
<ImageView
    android:id="@+id/presence_icon"
    android:layout_width="@dimen/icon_size"
    android:layout_height="@dimen/icon_size"
    android:layout_marginRight="@dimen/margin_small"
    ...
    />
{% endhighlight %}

By defining such dimensional values in `layout.xml`, you ensure consistent alignment and spacing across screens. This gives your application a tighter look and a more professional appearance.

## Colors

In a file named `colors.xml`, we can define color values like so:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="app_text">#464646</color>
    <color name="app_text_dark">#2E2E2E</color>
    <color name="app_warning">#E45620</color>
</resources>
{% endhighlight %}

Colors defined in `colors.xml` are helpful when we must change the color of a view based on data known only at run-time. For example, the following code changes the color of the displayed credits to `app_warning` if their number is below a certain threshold. It selects this color through the `getColor` method of a `Resources` object, which is returned from the `getResources()` method of an `Activity` instance:

{% highlight java %}
if (creditsRemaining < CREDITS_WARNING_THRESHOLD) {
  int warningColor = getResources().getColor(R.color.app_warning);
  creditsRemainingView.setTextColor(warningColor);
}
{% endhighlight %}

We can also reference these values in a layout XML file using the prefix `@color`:

{% highlight xml %}
<TextView
    android:id="@+id/title"
    android:textColor="@color/app_text_dark"
    android:textStyle="bold"
    ...
    />
{% endhighlight %}

Doing this is less common. The primary colors of your app should be [defined in a theme](http://developer.android.com/guide/topics/ui/themes.html#ApplyingStyles) and therefore applied automatically.

## Plurals

Finally, let's touch on a useful feature of `strings.xml`: You can define plural forms of strings, thereby avoiding embarrassing grammar mistakes like "one credits" in your application. In fact, this singleton form is the only irregularity we must guard against in the English language:

* zero credits
* one *credit*
* two credits, three credits, etc.

Instead of defining a simple `string` element in `strings.xml`, we define a `plurals` element with `item` child elements. Each `item` child has a `quantity` attribute specifying TODO. The `other` quantity is like the `default` label of a `switch` construct, and is used when . In English, we only need the quantities `"one"` and `"other"`:

{% highlight xml %}
<string name="shop_title">Purchase upgrade</string>
<plurals name="shop_prompt">
  <item quantity="one">Purchasing %1$s costs %2$d credit</item>
  <item quantity="other">Purchasing %1$s costs %2$d credits</item>
</plurals>
{% endhighlight %}


In a Java source file, we select an appropriate plural string through the `getQuantityString` method of a `Resources` object. For example, given a `ShopItem` instance with a string `title` field and an integer `credits` field:

{% highlight java %}
String prompt = getResources().getQuantityString(
    R.plurals.shop_prompt, shopItem.credits, shopItem.title, shopItem.credits);
{% endhighlight %}

Note that while we typically select a `string` element with `R.string`, we select a `plurals` element with `R.plurals`, as we do here with `R.plurals.shop_prompt`. The second parameter, `shopItem.credits`, is the integer value evaluated to select the appropriate plural string. The third and fourth parameters, `shopItem.title` and `shopItem.credits`, are the values substituted into the selected string. In the case above:

* If `shopItem.title` equals `"Product A"` and `shopItem.credits` equals `1`, then `getQuantityString` selects the string defined by the `item` element with `quantity="one"` . After substituting , `"Purchasing Product A costs 1 credit"`.
* If `shopItem.title` equals `"Product B"` and `shopItem.credits` equals `5`, then `getQuantityString` selects the string defined by the `item` element with `quantity="other"`.

In addition to `"one"` and `"other"`, other recognized `quantity` values are `"zero"`, `"two"`, `"few"`, and `"many"`. The use of these values is required when the language requires special treatment of the number 0, any number containing 2, any "small" number, or any "large" number. If you are not targeting a language other than English, [consult the official documentation](http://developer.android.com/guide/topics/resources/string-resource.html#Plurals).

### Dealing with irregular formats

Note that all quantity strings contained in a `plurals` element must contain the same format specifiers in the same order. For example, you cannot do:

{% highlight xml %}
<plurals name="shop_prompt">
  <item quantity="zero">%1$s is free!</item>
  <item quantity="one">Purchasing %1$s costs %2$d credit</item>
  <item quantity="other">Purchasing %1$s costs %2$d credits</item>
</plurals>
{% endhighlight %}

If `shopItem.credits` equals `0`, then `getQuantityString` will throw an exception at runtime because the second string formatting argument, `shopItem.credits`, has no corresponding format specifier `%2$d` in the string. Instead, this string should be moved to its own `string` element:

{% highlight xml %}
<string name="shop_prompt_zero">%1$s is free!</item>
<plurals name="shop_prompt">
  <item quantity="one">Purchasing %1$s costs %2$d credit</item>
  <item quantity="other">Purchasing %1$s costs %2$d credits</item>
</plurals>
{% endhighlight %}

You can use a `switch` construct to use this string when `shopItem.credits` equals `0`:

{% highlight java %}
String prompt = null;
switch (shopItem.credits) {
case 0:
  prompt = getString(R.string.shop_prompt_zero, shopItem.title);
  break;
default:
  prompt = getResources().getQuantityString(
      R.plurals.shop_prompt, shopItem.credits, shopItem.title, shopItem.credits);
}
{% endhighlight %}

