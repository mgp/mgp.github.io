---
layout: post
title:  "Additional Android Resource Types"
tags: ["android", "resources"]
---

Most Android developers associate the `values` resources folder with simple `string` elements in `strings.xml`. But [buried in the Android documentation](http://developer.android.com/guide/topics/resources/more-resources.html) are some additional resource types that not only simplify development, but help make your application look professional. Let's look at a few.

## Dimensions

In a file named `layout.xml`, we can define dimension values using the `dimen` element:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <dimen name="margin_large">24dp</dimen>
    <dimen name="margin_small">12dp</dimen>
    <dimen name="button_small_height">36dp</dimen>
    <dimen name="button_large_height">52dp</dimen>
    <dimen name="icon_size">32dp</dimen>
</resources>
```

Note that we're using `dp`, or [density-independent pixels](http://developer.android.com/guide/topics/resources/more-resources.html#Dimension), instead of `sp`. It is recommended that you use the latter only when specifying font sizes, and the former in all other cases. Once we've defined these values, we can reference them in layout XML files by using the prefix `@dimen`. For example, the following `ImageView` element in a `RelativeLayout` defines its width and height as `icon_size`, and its right margin as `margin_small`:

```xml
<ImageView
    android:id="@+id/presence_icon"
    android:layout_width="@dimen/icon_size"
    android:layout_height="@dimen/icon_size"
    android:layout_marginRight="@dimen/margin_small"
    ...
    />
```

By defining such dimensional values in `layout.xml`, you ensure consistent alignment and spacing across screens. This gives your application a tighter look and a more professional appearance.

For additional information on `dimen` elements, see [the Android documentation](http://developer.android.com/guide/topics/resources/more-resources.html#Dimension).

## Colors

In a file named `colors.xml`, we can define color values using the `color` element:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="app_text">#464646</color>
    <color name="app_text_dark">#2E2E2E</color>
    <color name="app_warning">#E45620</color>
</resources>
```

Colors defined in `colors.xml` are helpful when we must change the color of a view based on data known only at run-time. For example, the following code changes the color of the displayed credits to `app_warning` if their number is below a certain threshold. It selects this color through the `getColor` method of a `Resources` object, which is returned from the `getResources()` method of an `Activity` instance:

```java
if (creditsRemaining < CREDITS_WARNING_THRESHOLD) {
  int warningColor = getResources().getColor(R.color.app_warning);
  creditsRemainingView.setTextColor(warningColor);
}
```

We can also reference these values in a layout XML file using the prefix `@color`:

```xml
<TextView
    android:id="@+id/title"
    android:textColor="@color/app_text_dark"
    android:textStyle="bold"
    ...
    />
```

Doing this is less common. The primary colors of your app should be [defined in a theme](http://developer.android.com/guide/topics/ui/themes.html#ApplyingStyles) and therefore applied automatically. And similar to dimensions, by defining such colors in `colors.xml`, you ensure consistent coloring across screens for a more professional appearance.

For additional information on `color` elements, see [the Android documentation](http://developer.android.com/guide/topics/resources/more-resources.html#Color).

## Plurals

Finally, let's touch on a useful feature of `strings.xml`: You can define plural forms of strings, thereby avoiding embarrassing grammar mistakes.

As a motivating example, consider the following `string` element in `strings.xml`:

```xml
<string name="shop_prompt">Purchasing %1$s costs %2$d credits</item>
```

Given a `ShopItem` instance with a string `title` field and an integer `credits` field, say we format this string by doing:

```java
String prompt = getString(
    R.string.shop_prompt, shopItem.title, shopItem.credits);
```

If `shopItem.title` equals `"Product A"` and `shopItem.credits` equals `1`, then this assigns the grammatically incorrect string `"Purchasing Product A costs 1 credits"` to `prompt`. In fact, the quantity of 1 is the only grammatical irregularity we must guard against in the English language:

* 0 credits
* 1 *credit*
* 2 credits, 3 credits, etc.

Knowing this, one solution is to define two `string` elements in `strings.xml`, where one is used for the quantity of 1, and the other is used for all other quantities:

```xml
<string name="shop_prompt_one">Purchasing %1$s costs %2$d credit</item>
<string name="shop_prompt_other">Purchasing %1$s costs %2$d credit</item>
```

We can then select and format the grammatically correct string by doing:

```java
String prompt = null;
if (shopItem.credits == 1) {
  prompt = getString(
      R.string.shop_prompt_one, shopItem.title, shopItem.credits);
} else {
  prompt = getString(
      R.string.shop_prompt_other, shopItem.title, shopItem.credits);
}
```

This solution, however, has several drawbacks:

* It litters your code with conditional constructs.
* It's not trivially obvious from `strings.xml` that `shop_prompt_one` and `shop_prompt_other` are alternative grammatical forms of the same string.
* It becomes unwieldy for other languages with more complicated grammatical rules for quantities. Examples include special treatment of 0 in Arabic, 2 in Welsh, and languages that distinguish "small" and "large" numbers.

### The `plurals` element

Using the `plurals` element addresses these drawbacks. It contains *quantity strings*, which are the different grammatical forms of a string for different quantities. Each quantity string is defined in an `item` element that is a child of the `plurals` element. A `quantity` attribute of each `item` element specifies for what quantities its quantity string should be used. For English, the only two relevant attribute values are `"one"` and `"other"`, but sometimes `"zero"` is used. The `"other"` value is like the `default` label of a `switch` construct, and is used when the given quantity does not match the `quantity` attribute belonging to any other `item` element.

We can rewrite our two `string` elements in `strings.xml` as:

```xml
<plurals name="shop_prompt">
  <item quantity="one">Purchasing %1$s costs %2$d credit</item>
  <item quantity="other">Purchasing %1$s costs %2$d credits</item>
</plurals>
```

In a Java source file, we select an appropriate quantity string through the `getQuantityString` method of a `Resources` object:

```java
String prompt = getResources().getQuantityString(
    R.plurals.shop_prompt, shopItem.credits, shopItem.title, shopItem.credits);
```

Note that while we typically select a `string` element with `R.string`, we select a `plurals` element with `R.plurals`, as we do here with `R.plurals.shop_prompt`. The second parameter, `shopItem.credits`, is the quantity evaluated to select the appropriate `item` element and its quantity string. The third and fourth parameters, `shopItem.title` and `shopItem.credits`, are the values substituted into the selected quantity string.

Above, if `shopItem.title` equals `"Product A"` and `shopItem.credits` equals `1`, then the `item` element with `quantity="one"` is selected, and `"Purchasing Product A costs 1 credit"` is assigned to `prompt`. But if `shopItem.credits` equals `5` instead, then the `item` element with `quantity="other"` is selected, and `"Purchasing Product A costs 5 credits"` is assigned to `prompt`.

Other recognized `quantity` values include `"two"`, `"few"`, and `"many"`. These are used in the aforementioned grammatical rules for languages other than English. Consult [the Android documentation](http://developer.android.com/guide/topics/resources/string-resource.html#Plurals) for more information.

### Dealing with irregular formats

Note that all quantity strings contained in a `plurals` element must contain the same format specifiers in the same order. For example, you cannot do:

```xml
<plurals name="shop_prompt">
  <item quantity="zero">%1$s is free!</item>
  <item quantity="one">Purchasing %1$s costs %2$d credit</item>
  <item quantity="other">Purchasing %1$s costs %2$d credits</item>
</plurals>
```

If `shopItem.credits` equals `0`, then `getQuantityString` will throw an exception at runtime because the second string formatting argument, `shopItem.credits`, has no corresponding format specifier `%2$d` in the string. Instead, this string should be moved to its own `string` element:

```xml
<string name="shop_prompt_zero">%1$s is free!</item>
<plurals name="shop_prompt">
  <item quantity="one">Purchasing %1$s costs %2$d credit</item>
  <item quantity="other">Purchasing %1$s costs %2$d credits</item>
</plurals>
```

Now select the string `shop_prompt_zero` when `shopItem.credits` equals `0`, and the correct quantity string from `shop_prompt` when `shopItem.credits` is any other value:

```java
String prompt = null;
if (shopItem.credits == 0) {
  prompt = getString(R.string.shop_prompt_zero, shopItem.title);
} else {
  prompt = getResources().getQuantityString(
      R.plurals.shop_prompt, shopItem.credits, shopItem.title, shopItem.credits);
}
```

For additional information on `plural` elements, see the [Android documentation](http://developer.android.com/guide/topics/resources/string-resource.html#Plurals).

