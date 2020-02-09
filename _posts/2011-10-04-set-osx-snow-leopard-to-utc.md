---
title: Set OSX Snow Leopard to UTC
---
I was recently looking into a production issue that had to do with time zones. In order to accurately debug it I had to set the time on my machine to UTC.

This is how I changed it on OSX Snow Leopard:

{% highlight shell %}
$ sudo ln -sf /usr/share/zoneinfo/UTC /etc/localtime
{% endhighlight %}

Once your done change it back:

{% highlight shell %}
$ sudo ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
{% endhighlight %}
