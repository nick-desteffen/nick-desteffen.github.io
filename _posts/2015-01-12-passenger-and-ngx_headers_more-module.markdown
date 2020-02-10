## Installing

[Nginx](http://nginx.org/) and [Phusion Passenger](https://www.phusionpassenger.com/) make a powerful combonation for hosting web applications. Nginx is very flexible and fast. Phusion Passenger is rock solid and can be used to host Ruby, Python, and Node.js applications all at once. Nginx out of the box unfortunally lacks the ability to set or modify HTTP headers from within the configuration files. This is easily rectified by adding the [ngx_headers_more](https://github.com/openresty/headers-more-nginx-module) module.

The easiest way to install Nginx with Passenger support is through the gem. You'll need to download and unarchive the **ngx_headers_more** module first, since we will need to point the installer to this location.

Below are the commands I use when installing on a server. If you are installing locally for development they will probably be a bit different, but not much. The **passenger-install-nginx-module** command is the important one. Note the **extra-configure-flags** option, this is how you can build Nginx with support for the module.

{% highlight shell %}
$ sudo su -
$ mkdir -p /opt/nginx/build-modules
$ wget -P /opt/nginx/build-modules https://github.com/openresty/headers-more-nginx-module/archive/v0.25.tar.gz
$ tar -xzf /opt/nginx/build-modules/v0.25.tar.gz -C /opt/nginx/build-modules
$ gem install passenger
$ passenger-install-nginx-module --prefix=/opt/nginx --extra-configure-flags="--add-module=/opt/nginx/build-modules/headers-more-nginx-module-0.25"
{% endhighlight %}

## Using

After you have installed Nginx with the **ngx_headers_more** module you will be able to add and remove headers as you please from within your Nginx configuration files. This can be done from within the http or location blocks. Below is a small example.

The **more_clear_headers** directive can be used to clear unnecessary headers, like the ones that Passenger and Nginx set to indicate their version (Server, X-Powered-By, X-Runtime).

The **more_set_input_headers** directive can be used to set a header on the incoming request.

The **more_set_headers** directive can be used to set a header on the outgoing response.

{% highlight conf %}
  http {
    more_clear_headers 'Server' 'X-Powered-By' 'X-Runtime';
    more_set_headers 'X-Current-Hostname: $remote_addr';
    location /foo {
      more_set_input_headers 'X-Location: Foo';
    }
  }
{% endhighlight %}


## Related Links

*   [Nginx](http://nginx.org/)
*   [Passenger Phusion](https://www.phusionpassenger.com/)
*   [Nginx Headers More Wiki](http://wiki.nginx.org/NginxHttpHeadersMoreModule)
*   [ngx_headers_more Github Page](https://github.com/openresty/headers-more-nginx-module)
