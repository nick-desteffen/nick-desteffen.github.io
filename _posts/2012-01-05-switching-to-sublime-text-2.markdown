#### Updated – 07/16/2012

Please see my **[updated post](http://www.nickdesteffen.com/blog/switching-to-sublime-text-2-updated)** for current instructions on getting Sublime Text 2 up and running on your machine.

#### Updated – 01/12/2012

I recently switched over to [Sublime Text 2](http://www.sublimetext.com/2) for my primary development editor. I was waiting to see what [TextMate 2](http://blog.macromates.com/2011/textmate-2-0-alpha/) would be like, needless to say I didn't see much an improvement. Sublime Text 2 has got some nifty features like multiple columns, rows, or even grid views. After a few hours of having 2 files open side by side I was hooked. Below are the things I had to do to get it customized enough for me to use every day. As I continue to customize and setup Sublime Text 2 I'll be updating this post.

## Command line Tool

To be able to open a file or folder from the command line from anywhere create a symlink to the **subl** command line tool.

{% highlight shell %}
ln -s "/Applications/Sublime Text 2.app/Contents/SharedSupport/bin/subl" /usr/bin/subl</div>
{% endhighlight %}

## RVM

Edit Sublime Text's ruby settings file:

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/Ruby/Ruby.sublime-build</div>
{% endhighlight %}

It should look like this:

{% highlight json %}
{
  "cmd": ["ruby", "$file"],
  "file_regex": "^(...*?):([0-9]*):?([0-9]*)",
  "selector": "source.ruby"
}
{% endhighlight %}

Change it to this (replacing nickd with your username) in order to use the rvm-auto-ruby binary:

{% highlight json %}{
  "cmd": ["/Users/nickd/.rvm/bin/rvm-auto-ruby", "$file"],
  "file_regex": "^(...*?):([0-9]*):?([0-9]*)",
  "selector": "source.ruby"
}
{% endhighlight %}

## Spacing and tabs

You can set Sublime Text 2 to use spaces instead of tabs and adjust the indentation, however it seems to only change it for the current file your on, to make it global edit the User Base File settings file. _<u>Update</u>_ – As Kevin Yank & Dimitar Dimitrov point out in the comments, it is best to put any custom user settings in the User packages. This way updates won't overwrite them.

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/User/Base\ File.sublime-settings
{% endhighlight %}

Add the following settings to the hash of options. You can take a peek at all the other package settings and put any customizations you want to retain here.

{% highlight javascript %}
  // The number of spaces a tab is considered equal to
  "tab_size": 2,

  // Set to true to insert spaces when tab is pressed
  "translate_tabs_to_spaces": true,
{% endhighlight %}

To open all packages to see what's in them:

{% highlight shell %}
$ subl ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/
{% endhighlight %}

## RSpec & Test::Unit

I found this great plugin for running both RSpec and Test::Unit tests from within Sublime Text 2, [https://github.com/maltize/sublime-text-2-ruby-tests](https://github.com/maltize/sublime-text-2-ruby-tests). It allows you to run all the tests in an individual file and also supports focused tests, however focused tests don't seem to work for me. To get this installed:

{% highlight shell %}
$ cd ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/
$ git clone https://github.com/maltize/sublime-text-2-ruby-tests.git
{% endhighlight %}

## Sublime Package Control

_Update_ – I'd like to thank Dimitar Dimitrov & Pablo Barrios for pointing out [Will Bond's](http://wbond.net/) Package Control. [Sublime Package Control](http://wbond.net/sublime_packages/package_control) is a great addition to Sublime's ecosystem. There are ton of useful packages and themes that you can add just by selecting them. Just follow the simple [installation instructions](http://wbond.net/sublime_packages/package_control/installation) and you'll be good to go.
Press **cmd + shift + p** and type "Package Control" to find all the options for this add on. You can also get to it under the **Preferences Menu**.

## CoffeeScript

Sublime Text 2 supports TextMate bundles, to get CoffeeScript syntax highlighting just install the bundle through the package control.

## Haml & Sass

To get Haml & Sass support clone [this repository](https://github.com/n00ge/sublime-text-haml-sass) and copy the Ruby Haml and SASS folders into the Packages folder.

{% highlight shell %}
$ git clone https://github.com/n00ge/sublime-text-haml-sass.git
$ open ~/Library/Application\ Support/Sublime\ Text\ 2/Packages/
{% endhighlight %}

## Related Links

*   [Sublime Text 2](http://www.sublimetext.com/2)
*   [Text Mate 2](http://blog.macromates.com/2011/textmate-2-0-alpha/)
*   [Sublime Text 2 Ruby Tests](https://github.com/maltize/sublime-text-2-ruby-tests)
*   [Sublime Package Control](http://wbond.net/sublime_packages/package_control)

