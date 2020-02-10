Since I wrote my [original post](http://www.nickdesteffen.com/blog/switching-to-sublime-text-2) on switching to [Sublime Text 2](http://www.sublimetext.com/) a lot has changed. The final version was released just a few weeks ago. I also got a new computer and had to reinstall it; what I had originally written is a bit out of date so here is an updated post.

## Command line Tool

To be able to open a file or folder from the command line from anywhere create a symlink to the **subl** command line tool.

{% highlight shell %}
$ sudo ln -s "/Applications/Sublime Text 2.app/Contents/SharedSupport/bin/subl" /usr/bin/subl
{% endhighlight %}

## RVM

Edit Sublime Text's ruby settings file:

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/Ruby/Ruby.sublime-build
{% endhighlight %}

It should look like this:

{% highlight javascript %}{
  "cmd": ["ruby", "$file"],
  "file_regex": "^(...*?):([0-9]*):?([0-9]*)",
  "selector": "source.ruby"
}
{% endhighlight %}

Change it to this (replacing nickd with your username) in order to use the rvm-auto-ruby binary:

{% highlight javascript %}
{
  "cmd": ["/Users/nickd/.rvm/bin/rvm-auto-ruby", "$file"],
  "file_regex": "^(...*?):([0-9]*):?([0-9]*)",
  "selector": "source.ruby"
}
{% endhighlight %}

## Preferences File

Sublime Text 2 stores all settings and preferences in editable files with key/value pairs. If you want to change any default settings edit the User Preferences settings file and place your changes there. This will prevent them from being overridden in an upgrade.

Edit the User Preferences file:

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/User/Preferences.sublime-settings
{% endhighlight %}

You can also edit this file from within Sublime Text. Click the menu item: **Sublime Text 2 => Preferences => Settings – User**. I changed mine to use spaces instead of tabs and adjust the indentation globally. I also turned off hot exit, word wrap, and a few other things. If you just look at the default settings file you can get an idea of what's available to change. Just copy the keys to your own user file and change the values. I recommend checking this file into [git](http://github.com) as well.

{% highlight javascript %}{
  "tab_size": 2,
  "translate_tabs_to_spaces": true,
  "word_wrap": "auto",
  "hot_exit": false
}
{% endhighlight %}

To open all packages to see what's in them:

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/
{% endhighlight %}

## Sublime Package Control

[Sublime Package Control](http://wbond.net/sublime_packages/package_control) is a great addition to Sublime's ecosystem. There are ton of useful packages and themes that you can add just by selecting them. Just follow the simple [installation instructions](http://wbond.net/sublime_packages/package_control/installation) and you'll be good to go.
Press **cmd + shift + p** and type "Package Control" to find all the options for this add on. You can also get to it under the **Preferences Menu**.

The packages you have installed are listed in another settings file. Any changes you make to this file will be automatically picked up by the package manager. It is easy to just add or remove packages here as well.

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/User/Package\ Control.sublime-settings
{% endhighlight %}

Some useful ones I've installed are: [CoffeeScript](https://github.com/jashkenas/coffee-script-tmbundle), [Dogs Colour Scheme](https://github.com/radiosilence/dogs-colour-scheme), [Haml](https://github.com/phuibonhoa/handcrafted-haml-textmate-bundle), [RSpec](https://github.com/SublimeText/RSpec), [RubyTest](https://github.com/maltize/sublime-text-2-ruby-tests), [Sass](https://github.com/nathos/sass-textmate-bundle), and [SCSS](http://sass-lang.com/). It's also a good idea to check this file into git.

## Sublime Text 2 Ruby Tests Bundler Integration

When you try running tests from within Sublime Text it will probably error out if you are using [Bundler](http://gembundler.com/). To fix this you need to override the command to execute the tests. Open you User Preferences file again and add:

{% highlight javascript %}
{
  // RubyTest settings
  "ruby_unit_exec": "bundle exec ruby",
  "ruby_rspec_exec": "bundle exec rspec"
}
{% endhighlight %}

## Sublime Text 2 Ruby Tests ANSI Color Fix

When running RSpec test from within Sublime Text 2 you'll notice that the output contains ANSI color codes.

To fix this you can edit this file:

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/Theme\ -\ Default/Widget.sublime-settings
{% endhighlight %}

and comment out the color_scheme line and add the one listed below. This doesn't break any highlighting and will cause the ANSI color codes to be interpreted correctly.

{% highlight javascript %}
{
  //"color_scheme": "Packages/Theme - Default/Widgets.stTheme"
  "color_scheme": "Packages/RubyTest/TestConsole.tmTheme"
}
{% endhighlight %}

This fix was posted on the [Github Issues page](https://github.com/maltize/sublime-text-2-ruby-tests/issues/33#issuecomment-3553701)

I hope this helps you out. If you have any suggestions or tips please post a comment!

## Related Links

*   [Sublime Text 2](http://www.sublimetext.com/)
*   [Sublime Text 2 Ruby Tests Color Fix](https://github.com/maltize/sublime-text-2-ruby-tests/issues/33#issuecomment-3553701)
*   [Sublime Package Control](http://wbond.net/sublime_packages/package_control)
