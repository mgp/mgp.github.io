---
layout: post
title:  "Placeholder Selectors and Mixins in Sass"
tags: ["sass", "mixins", "selectors"]
---

The [Sass language](http://sass-lang.com/) describes a *mixin* as allowing you to "re-use whole chunks of CSS, properties or selectors." But using mixins to promote reuse in a Sass source file does not eliminate repetition from the generated CSS file and reduce its size. As we will see, to reap those benefits we must use *placeholder selectors*.

## Using mixins

To illustrate that mixins do not eliminate repetition from the generated CSS file, consider the following mixin named `section-list`:

```scss
@mixin section-list {
  ul {
    list-style-type: none;
    li p {
      color: #444;
      margin-top: 0.25em;
    }
  }
}
```

And consider its inclusion later by the selectors `#main.code` and `#main.talks`:

```scss
#main {
  &.code {
    @include section-list;
    padding-top: 0.5em;
    li .title {
      font-weight: bold;
    }
  }

  &.talks {
    @include section-list;
    li img {
      float: left;
    }
  }
}
```

Together they generate the following CSS:

{% highlight css linenos %}
#main.code {
  padding-top: 0.5em; }
  #main.code ul {
    list-style-type: none; }
    #main.code ul li p {
      color: #444;
      margin-top: 0.25em; }
  #main.code li .title {
    font-weight: bold; }
#main.talks ul {
  list-style-type: none; }
  #main.talks ul li p {
    color: #444;
    margin-top: 0.25em; }
#main.talks li img {
  float: left; }
{% endhighlight %}

The style declarations belonging to the mixin, which appear only once in the Sass source file, appear twice in the generated CSS. On lines 4 and 11, both `#main.code ul` and `#main.talks ul` declare:

```css
list-style-type: none;
```

While on lines 6--7 and 13--14, both `#main.code ul li p` and `#main.talks ul li p` declare:

```css
color: #444;
margin-top: 0.25em;
```

While this isn't much repetition, we can easily imagine `section-list` encompassing more than three declarations and this repetition growing. I deliberately kept `section-list` simple for illustrative purposes.

## Using placeholder selectors

Now consider `section-list` converted to a placeholder selector. The name of such a selector is prefixed with `%`:

```scss
%section-list {
  ul {
    list-style-type: none;
    li p {
      color: #444;
      margin-top: 0.25em;
    }
  }
}
```

This placeholder selector can only be used with the `@extend` directive, and so such a selector is also called an *extend-only selector*. Below, the selectors `#main.code` and `#main.talks` extend `%section-list`:

```scss
#main {
  &.code {
    @extend %section-list;
    padding-top: 0.5em;
    li .title {
      font-weight: bold;
    }
  }

  &.talks {
    @extend %section-list;
    li img {
      float: left;
    }
  }
}
```

Together they generate the following CSS:

```css
#main.code ul, #main.talks ul {
  list-style-type: none; }
  #main.code ul li p, #main.talks ul li p {
    color: #444;
    margin-top: 0.25em; }

#main.code {
  padding-top: 0.5em; }
  #main.code li .title {
    font-weight: bold; }
#main.talks li img {
  float: left; }
```

The style declarations belonging to the placeholder selector appear only once in the Sass source file *and* the generated CSS, eliminating the repetition seen earlier. Consequently, this CSS is shorter than the CSS generated from using the mixin, containing just 272 characters on 12 lines instead of 355 characters on 16 lines. Also note that the style declarations of the placeholder selector are separated from those exclusive to the selectors `#main.code` and `#main.talks` by a blank line, thereby enhancing its readability and creating a form similar to its Sass source file.

### General selector inheritance

Placeholder selectors described above are a variant of *selector inheritance*. With selector inheritance, one class selector extends another class selector, and all style declarations belonging to the former are inherited by the latter. Both the parent and child class selectors are rules in the generated CSS. This is unlike using a placeholder selector, which generates no rule in the CSS. (For example, above, there was no selector that contained the placeholder selector name `section-list`.)

As an example of selector inheritance, consider the following selector `.photo-caption` extending selector `.caption`:

```css
.caption {
  font-style: italic;
  font-size: 0.75em;
}
.photo-caption {
  @extend .caption;
  padding-top: 0.5em;
}
```

This compiles to the following CSS:

```css
.caption, .photo-caption {
  font-style: italic;
  font-size: 0.75em; }

.photo-caption {
  padding-top: 0.5em; }
```

If `.photo-caption` did not extend `.caption`, then HTML must use `class="caption photo-caption"` instead of simply `class="photo-caption"` to apply the same style declarations.

For more information, consult the [Sass reference for `@extend`](http://sass-lang.com/docs/yardoc/file.SASS_REFERENCE.html#extend).

### When to use mixins

Mixins are still useful, however, but not in the simple form above that can be replaced by selector inheritance.

First, mixins can accept arguments. For example, the following mixin named `aside` accepts two, named `$bg-color` and `$border-color`. The mixin uses the former to define the `background-color` property, and the latter to define the `border` property:

```scss
@mixin aside($bg-color, $border-color) {
  background-color: $bg-color;
  border: 3px solid $border-color;
}
```

Below, the class selector `.info-aside` calls this mixin:

```scss
.info-aside {
  @include aside(#f5f5ff, #e5e5ff);
}
```

Together they generate the following CSS:

```css
info-aside {
  background-color: #f5f5ff;
  border: 3px solid #e5e5ff; }
```

Second, you can replace the `@content` directive of a mixin with declarations that are passed to the mixin. This is similar to [template inheritance](http://www.smarty.net/inheritance) found in many HTML template engines. The following snippets come from the Sass reference. Given this mixin:

```scss
@mixin apply-to-ie6-only {
  * html {
    @content;
  }
}
```

The following substitutes the nested `#logo` rule for the `@content` directive above. Note that `@include` and the mixin name precede the opening brace:

```scss
@include apply-to-ie6-only {
  #logo {
    background-image: url(/logo.gif);
  }
}
```

Together they generate the following CSS:

```css
* html #logo {
  background-image: url(/logo.gif);
}
```

But if a mixin does not accept any arguments and does not contain a `@content` directive, replace it with a placeholder selector.

