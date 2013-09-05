---
layout: post
title:  "Sass Mixins are Not Lightweight"
tags: ["sass", "mixins", "selectors"]
---

The [Sass language](http://sass-lang.com/) describes a *mixin* as allowing you to "re-use whole chunks of CSS, properties or selectors." Given this, you might TODO. But as we will see, while a mixin eliminates repetition from a Sass source file, only *selector inheritance* eliminates repetition from the generated CSS.

## Using mixins

Consider the following mixin `section-list`, using the `@mixin` directive:

```css
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

And consider its inclusion later in the file, using the `@include` directive:

```css
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

When combined together, they generates the following CSS:

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

This contains repetition that was eliminated from the Sass source file. On lines 4 and 11, both `#main.code ul` and `#main.talks ul` use:

```css
list-style-type: none;
```

While on lines 6--7 and 13--14, both `#main.code ul li p` and `#main.talks ul li p` use:

```css
color: #444;
margin-top: 0.25em;
```

While this isn't much repetition, it's not hard to imagine `section-list` declaring more than three styles and this repetition growing. I deliberately kept `section-list` simple for illustrative purposes.

## Using selector inheritance

Now consider `section-list` converted to a TODO. Its name is now prefixed with `%`:

```css
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

And consider its extension later in the file, using `@extend`:

```css
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

This generates the following CSS:

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

This eliminates the repetition seen earlier. In fact, these styles belonging to the TODO and shared by multiple selectors are even separated from the styles  by a new line, enhancing their readability.

It's also shorter, containing just 272 characters on 12 lines, as opposed to 355 characters on 16 lines.

### Abstract styles

Note that when using selector inheritance above with `%section-list`, there was no . `%section-list` was like an abstract TODO, while `#main.code ul`, `main.talks ul`, and so forth were its concrete implementations. Consider the following Sass code from its reference:

```css
.error {
  border: 1px #f00;
  background-color: #fdd;
}
.seriousError {
  @extend .error;
  border-width: 3px;
}
```

Note that TODO `seriousError` extends `.error`.

This is compiled to the following CSS:

```css
.error, .seriousError {
  border: 1px #f00;
  background-color: #fdd;
}

.seriousError {
  border-width: 3px;
}
```

If `.seriousError` did not use `@extend .error`, then a client must use `class="error seriousError"` instead of simply `class="seriousError"` to use the same styles.

For more information, consult the [Sass reference for `@extend`](http://sass-lang.com/docs/yardoc/file.SASS_REFERENCE.html#extend).

### When to use mixins

Mixins are still useful, however, but not in the simple form above that can be replaced by selector inheritance.

First, mixins can accept arguments. For example, the following mixin TODO.

```css
```

This can be instantiated by using:

```css
```

This generates the CSS:

```css
```

Second, you can replace the `@content` directive of a mixin with a block of styles that are passed to the mixin. This is similar to [template inheritance](http://www.smarty.net/inheritance) that is now common among HTML template engines. For example, TODO.

```css
```

The following styles are substituted for the `@content` directive. Note that `@include` and the mixin name now precedes the opening brace:

```css
```

This generates the CSS:

```css
```

But if a mixin does not accept any arguments or a content block, use selector inheritance.

