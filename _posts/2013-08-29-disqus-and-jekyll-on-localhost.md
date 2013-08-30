---
layout: post
title:  "Disqus and Jekyll on localhost"
tags: ["disqus", "jekyll", "localhost"]
---

I recently added [Disqus](http://disqus.com/) comments to every blog post by adding its [universal code](http://disqus.com/admin/universalcode/) to the bottom of the [Jekyll](http://jekyllrb.com/) file `_layout/post.html`. Upon logging into Disqus, however, I noticed that many discussion URLs began with `http://localhost:4000/`, such as `http://localhost:4000/2013/08/28/additional-android-resource-types/`. Other discussion URLs had the same path but began with `http://omgitsmgp.com/`, such as `http://omgitsmgp.com/2013/08/28/additional-android-resource-types/`.

Discussion URLs beginning with `http://localhost:4000/` were created when I served the site locally, usually to preview a [draft](http://jekyllrb.com/docs/drafts/) generated with the flag `--drafts`. Discussion URLs beginning with `http://omgitsmgp.com/` were created when [GitHub Pages](http://pages.github.com/) later served the site publicly.

After reading through the [Jekyll documentation](http://jekyllrb.com/docs/home/), I had come up with several working solutions that were bad, but none that were elegant. I then [created an issue](https://github.com/mojombo/jekyll/issues/1470) to request a feature that could enable one elegant solution; its [first reply](https://github.com/mojombo/jekyll/issues/1470#issuecomment-23472470) highlighted an oversight in the documentation: Jekyll supports using multiple configuration files. This was now my elegant solution. (A [pull request](https://github.com/mojombo/jekyll/pull/1474) to clarify the documentation has since been merged.)

## Using multiple configuration files

By supporting multiple configuration files, Jekyll can use different variables when serving locally or serving publicly. In this case, we want a boolean site variable named `enable_disqus` to control whether or not Disqus comments are enabled. This variable should be `true` when serving publicly, and `false` when serving locally.

My `_config.yml` file contains the following setting to enable Disqus comments on blog posts:

```yaml
enable_disqus: true
```

While my `_local_config.yml` file contains the following setting to disable these Disqus comments:

```yaml
enable_disqus: false
```

In the file `_layout/post.html`, I conditionally include the Disqus universal code:

```html
{% raw %}{% if site.enable_disqus %}{% endraw %}
  <!-- Universal code goes here... -->
{% raw %}{% endif %}{% endraw %}
```

Now when I serve the site locally, I pass the flag `--config=_config.yml,_local_config.yml`:

```bash
jekyll serve --watch --drafts --config=_config.yml,_local_config.yml
```

When I serve the site locally with this flag, the `enable_disqus` value of `false` defined in `_local_config.yml` overrides the value of `true` defined in `_config.yml`. So the Disqus universal code is not included, and no discussion URLs beginning with `http://localhost:4000/` can be created. When GitHub Pages serves the site publicly, only `_config.yml` is used, and so `enable_disqus` is `true`. So the Disqus universal code is included, and discussion URLs beginning with `http://omgitsmgp.com/` can be created.

In the future I may wish to disable additional features when serving locally while enabling those features when serving publicly, or vice versa. This simply requires defining additional variables in `_config.yml` and overriding their values in `_local_config.yml`. An alternative is to define a single variable named `serving_publicly`, with a value of `true` in `_config.yml` and `false` in `_local_config.yml`. Then the file `_layout/post.html` would instead contain:

```html
{% raw %}{% if site.serving_publicly %}{% endraw %}
  <!-- Universal code goes here... -->
{% raw %}{% endif %}{% endraw %}
```

From looking at this code, we can now tell that serving publicly enables Disqus. But from looking at `_local_config.yml`, we can no longer tell what site features are toggled when switching between serving locally and serving publicly. It is a trade-off.

