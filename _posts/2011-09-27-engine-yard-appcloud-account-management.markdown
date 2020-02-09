If you are working on several projects that are all deployed on [EngineYard AppCloud](http://www.engineyard.com/products/appcloud) managing deployments can be a bit tricky.

When you first deploy an application using the [engineyard gem](https://github.com/engineyard/engineyard) you authenticate using your email and password. A file named **.eyrc** containing an API token is then generated in your home directory. Each deployment the gem reads the environment variable **ENV['EYRC']**, this contains the location of the .eyrc file. EngineYard [recommends](http://docs.engineyard.com/deployment-faqs.html) renaming the .eyrc file and using aliases to change update the environment variable with the location of the renamed file for account you are deploying.

My solution to this problem was to move the .eyrc right into the source code. I then wrote a rake task to handle deployments. Each task in the Rakefile first calls **:set_eyrc** which just updates the environment variable with with the path of the .eyrc file stored in the project.

With this approach everybody working on your project can deploy without setting up aliases on their machine (provided their public key is on the server being deployed to).

Another thing I did to ease deployments was move our Chef Deploy recipes into the project under the directory **/ey-cloud-recipes**, this way it is versioned and can be updated using the same system for multiple accounts.

**/config/.eyrc**

{% highlight yaml %}
---
api_token: YOUR_API_TOKEN
{% endhighlight %}

I've included the Rakefile below.
<script src="https://gist.github.com/1228245.js?file=engineyard.rake"></script>

## Related Links

*   [EngineYard AppCloud](http://www.engineyard.com/products/appcloud)
*   [EngineYard CLI User Guide](http://docs.engineyard.com/ey_cli_user_guide.html)
*   [EngineYard Deployment FAQ](http://docs.engineyard.com/deployment-faqs.html)
*   [EngineYard Gem](https://github.com/engineyard/engineyard)
