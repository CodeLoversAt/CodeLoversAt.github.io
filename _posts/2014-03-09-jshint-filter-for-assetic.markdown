---
layout: post
status: publish
published: true
title: "JsHint filter for Assetic"
author:
  display_name: d
  login: admin
  email: daniel@codelovers.at
  url: http://d.velopment.at
author_login: admin
author_email: daniel@codelovers.at
author_url: http://d.velopment.at
wordpress_id: 7
wordpress_url: http://blog.codelovers.at/?p=7
date: '2014-03-09 12:30:05 +0100'
date_gmt: '2014-03-09 12:30:05 +0100'
categories: Symfony PHP Assetic JsHint
comments: []
---
Yesterday at the university we were introduced to [gulp.js](http://gulpjs.com/). In an example we created some tasks for concatenating and minifying/uglifying JavaScript assets – quite the usual stuff you’ll use [Assetic](https://github.com/kriswallsmith/assetic) for in your Symfony projects.

But we also created a task which piped the assets through [JsHint](http://www.jshint.com/) to perform code (quality) analysis. I really liked the idea of having an analysis run on your code on compilation time, so I looked for an existing JsHint filter for Assetic. Since I couldn’t find any (in an admittedly short Google search), I decided to quickly implement one on my own:

[https://github.com/CodeLoversAt/assetic-jshint](https://github.com/CodeLoversAt/assetic-jshint) will pass the given asset through JsHint and throw a `FilterException` if any errors occurr. Since you might be interested in the actual file that produced the error, this filter will only work with a temporary file if it cannot determine the asset’s file name (like if you pass in an instance of a `StringAsset`).

As I mentioned in the first paragraph, I thought of using this filter in our Symfony projects from the beginning, so unsurprisingly I’ve also created a Symfony bundle which registers that filter:

[https://github.com/CodeLoversAt/assetic-jshint-bundle](https://github.com/CodeLoversAt/assetic-jshint-bundle)

Here’s some example usage:

Lets consider the following two scripts. The first is called `valid.js` and it contains – as you might have guessed already – some simple, valid JavaScript

{% highlight javascript %}
(function() {
    "use strict";
    alert('I am valid!');
})();
{% endhighlight %}

The second one is called invalid.js and there we’ve forgotten a semicolon.

{% highlight javascript %}
(function() {
    "use strict";
    alert('I am invalid because I\'m missing a semicolon at the end of this line!')
})();
{% endhighlight %}


Now we want to use this scripts in a template. First the valid one:

{% highlight html %}
{% raw %}
{% javascripts
'@CodeLoversDemoBundle/Resources/public/js/valid.js'
output='compiled/js/demo_script.js' filter='jshint'
%}
<script type="text/javascript" src="{{ asset_url }}"></script>
{% endjavascripts %}
{% endraw %}
{% endhighlight %}

When we run `app/console assetic:dump` now, we won’t notice any difference, since the filter doesn’t have anything to complain about:

```bash
web65@srv:~/symfony$ app/console assetic:dump --env=prod --no-debug
Dumping all prod assets.
Debug mode is off.
13:26:13 [file+] /var/www/clients/client1/web65/symfony/app/../web/compiled/js/demo_script.js
web65@srv:~/symfony$
```

Now let’s see what happens if we add the invalid file to that asset:

{% highlight html %}
{% raw %}
{% javascripts
'@CodeLoversDemoBundle/Resources/public/js/valid.js'
'@CodeLoversDemoBundle/Resources/public/js/invalid.js'
output='compiled/js/demo_script.js' filter='jshint'
%}
<script type="text/javascript" src="{{ asset_url }}"></script>
{% endjavascripts %}
{% endraw %}
{% endhighlight %}

Again, we dump the assets, and here we go: a beautiful error appears:

```bash
web65@srv:~/symfony$ app/console assetic:dump --env=prod --no-debug
Dumping all prod assets.
Debug mode is off.
13:26:58 [file+] /var/www/clients/client1/web65/symfony/app/../web/compiled/js/demo_script.js
  [Assetic\Exception\FilterException]
  An error occurred while running:
  '/usr/local/bin/jshint' '/var/www/clients/client1/web65/symfony/src/CodeLov
  ers/DemoBundle/Resources/public/js/invalid.js'
  Output:
  /var/www/clients/client1/web65/symfony/src/CodeLovers/DemoBundle/Resources/
  public/js/invalid.js: line 3, col 84, Missing semicolon.
  1 error
  Input:
  // some invalid JavaScript code
  (function() {
      alert('I am invalid because I\'m missing a semicolon at the end of this
   line!')
  })();
assetic:dump [--watch] [--force] [--period="..."] [write_to]
web65@srv:~/symfony$
```

As you can see, the error contains the real file name, where the error occurred. This should be quite helpful in real life scenarios.
