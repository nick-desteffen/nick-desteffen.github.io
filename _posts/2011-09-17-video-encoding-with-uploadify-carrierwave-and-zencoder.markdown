For a recent Rails 3.1 project one of the features requested was to allow users to painlessly upload videos and have them all look consistent. We choose [Zencoder](http://zencoder.com) for our video encoding service and [Rackspace Cloud Files](http://www.rackspace.com/cloud/cloud_hosting_products/files/) for storage. We were already using the gem [CarrierWave](https://github.com/jnicklas/carrierwave) for attachments and the jQuery plugin [Uploadify](http://www.uploadify.com/) for uploading files.

Having large files uploaded through Uploadify works really well. As soon as a file is selected, it starts to upload; this is a much better experience than making a user wait while a video uploads over a standard request. This post will outline all the steps to get a video uploaded, encoded, and displayed to your users.

This is a short rundown of what will happen:

1.  User uploads video using Uploadify
2.  Carrierwave uploads the video to Rackspace
3.  After upload is complete a video encoding job is submitted to Zencoder
4.  Zencoder retrieves the uploaded video file from Rackspace
5.  Zencoder encodes the video and replaces the uploaded file on Rackspace
6.  Zencoder calls back to the server to notify that the job has completed
7.  Encoded video is now available to user

The complete source code for this blog post is available on my Github account. You can just update it with your own Zencoder and Rackspace account credentials and it should work.

[https://github.com/nick-desteffen/carrierwave_zencoder_example](https://github.com/nick-desteffen/carrierwave_zencoder_example)

## Create project

Start out with a fresh Rails 3.1 project.

{% highlight shell %}
$ rails new carrierwave_zencoder_example
$ rm carrierwave_zencoder_example/public/index.html
{% endhighlight %}

## Add Javascript and Gem dependencies

Lets get all the dependencies out of the way first.

1. [Download Uploadify](http://www.uploadify.com/download/) and extract it. The version we used was 2.1.4\. Put the extracted folder under */vendor/assets/javascripts*
1. Update your */app/assets/stylesheets/application.css* manifest file to include the Uploadify stylesheet:

{% highlight css %}
*= require_self
*= require_tree .
*= require jquery.uploadify-v2.1.4/uploadify
{% endhighlight %}

{:start="3"}
1. Update your */app/assets/javascripts/application.js* manifest file to include all the Uploadify javascript files

{% highlight javascript %}
//= require jquery
//= require jquery_ujs
//= require_tree .
//= require jquery.uploadify-v2.1.4/jquery.uploadify.v2.1.4.min
//= require jquery.uploadify-v2.1.4/swfobject
{% endhighlight %}

{:start="4"}
1. Add the following gems to your *Gemfile*

{% highlight ruby %}
gem "carrierwave"
gem "fog"
gem "zencoder"
{% endhighlight %}

{:start="5"}
1. Run:

{% highlight shell %}
$ bundle install
{% endhighlight %}

## Configure Rackspace & CarrierWave

Now that we have CarrierWave installed we need to configure it to use Rackspace to store uploaded videos. CarrierWave uses the gem called [Fog](https://github.com/geemus/fog) to manage remote file storage. This allows CarrierWave to use any storage backend. Fog has support for Amazon S3 and Google Storage for Developers as well, either of those options can easily be substituted.

I'm going to assume you have already signed up for Rackspace Cloud Files:

1. Log in and click on Your Account and then API Access and make note of your API key.
2. Add a container called **blog.uploads** (or whatever your in the mood for).
3. Click on the container you created in the listing and you should be able to change some properties:
    1. Check the box **Publish to CDN**.
    2. There should now be a CDN URL listed, make note of that.
4. CarrierWave is best configured using a Rails initalizer. Under /config/initializers create a new file named carrierwave.rb and put the following code into it.

*/config/initalizers/carrierwave.rb*

{% highlight ruby %}
CarrierWave.configure do |config|
  if Rails.env.test?
    config.storage = :file
    config.enable_processing = true
  else
    config.storage = :fog
    config.fog_credentials = {
      :provider           => 'Rackspace',
      :rackspace_username => 'YOUR_RACKSPACE_USERNAME',
      :rackspace_api_key  => 'YOUR_RACKSPACE_API_KEY'
    }
    config.fog_directory = 'blog.uploads'
    config.fog_host = 'YOUR_RACKSPACE_CDN_URL'
  end
end
{% endhighlight %}

(Update it with the username you used when signing up for Rackspace Cloud Files, the API key fro your account. Under *config.fog_host* put the CDN URL.)

If you want files served over SSL just update the URL's second subdomain to be **https**.
Example:

* Over HTTP: http://c293023.r87.cf2.rackcdn.com
* Over HTTPS: https://c744687.r87.ssl.rackcdn.com


## Configure Zencoder

I'm going to assume you have a Zencoder account.

1. Login and click API. Take note of the key.
2. Create a Rails initalizer (*/config/initalizers/zencoder.rb*) for zencoder.

{% highlight ruby %}
Zencoder.api_key = 'YOUR_API_KEY'
{% endhighlight %}

## Add Rack middleware to allow file uploads through flash

Rails has built in cross site request forgery protection. When a form is submitted a token is sent along with it. Since Uploadify is flash based we need to create a Rack middleware that tacks this parameter on to the request from flash.

I followed the instructions in this [post](http://www.glrzad.com/ruby-on-rails/using-uploadify-with-rails-3/) by [Damian Galarza](http://www.glrzad.com/) on how to do this.
He gives a very thorough explanation of uploading to Rails using Uploadify, I'm gonna shorten it a bit to just include all the code that's needed to get it up and running.

1. Create a a middleware folder under /app and a new file named flash_session_cookie_middleware.rb
1. Here is the middleware code:
    */app/middleware/flash_session_cookie_middleware.rb*

{% highlight ruby %}
require 'rack/utils'

class FlashSessionCookieMiddleware
  def initialize(app, session_key = '_session_id')
    @app = app
    @session_key = session_key
  end

  def call(env)
    if env['HTTP_USER_AGENT'] =~ /^(Adobe|Shockwave) Flash/
      req = Rack::Request.new(env)
      env['HTTP_COOKIE'] = [ @session_key,
                             req.params[@session_key] ].join('=').freeze unless req.params[@session_key].nil?
      env['HTTP_ACCEPT'] = "#{req.params['_http_accept']}".freeze unless req.params['_http_accept'].nil?
    end

    @app.call(env)
  end
end
{% endhighlight %}

{:start="3"}
1. Update your (*config/application.rb*) and tell it to autoload the middleware.

{% highlight ruby %}
config.autoload_paths += %W(#{config.root}/app/middleware/**/*)
{% endhighlight %}

{:start="4"}
4. Edit your session_store (*/config/initalizers/session_store.rb*) initializer to include middleware.

{% highlight ruby %}
Rails.application.config.middleware.insert_before(
  ActionDispatch::Session::CookieStore,
  FlashSessionCookieMiddleware,
  Rails.application.config.session_options[:key]
)
{% endhighlight %}

## Create a Video model

Create a model to store our uploaded video information. It will have an attachment column for the filename used by CarrierWave, a column to put the output id from Zencoder and a processed boolean flag.

1.  Create Model
2.  Migrate your database
3.  In your *video.rb* model add a method to update the processed flag.

{% highlight bash %}
$ bundle exec rails g model video attachment:string zencoder_output_id:string processed:boolean
$ bundle exec rake db:migrate
{% endhighlight %}

{% highlight ruby %}
class Video < ActiveRecord::Base

  attr_accessible :attachment

  validates_presence_of :attachment

  def processed!
    update_attribute(:processed, true)
  end

end
{% endhighlight %}

## Create Zencoder callback controller

This controller will handle the callback from Zencoder when an encoding job is complete. Zencoder sends data as JSON.

1. Generate controller
{% highlight bash %}
$ bundle exec rails g controller zencoder_callback
{% endhighlight %}

{:start="2"}
1. Update routes (*/config/routes.rb*)
{% highlight ruby %}
post "zencoder-callback" => "zencoder_callback#create", :as => "zencoder_callback"
{% endhighlight %}

{:start="3"}
1. Update controller (*/app/controllers/zencoder_callback_controller.rb*)
{% highlight ruby %}
class ZencoderCallbackController < ApplicationController

  skip_before_filter :verify_authenticity_token

  def create
    zencoder_response = ''
    sanitized_params = sanitize_params(params)
    sanitized_params.each do |key, value|
      zencoder_response = key.gsub('\"', '"')
    end

    json = JSON.parse(zencoder_response)
    output_id = json["output"]["id"]
    job_state = json["output"]["state"]

    video = Video.find_by_zencoder_output_id(output_id)
    if job_state == "finished" && video
      video.processed!
    end

    render :nothing => true
  end

  private

  def sanitize_params(params)
    params.delete(:action)
    params.delete(:controller)
    params
  end

end
{% endhighlight %}

#### Whats going on in the controller?
1.  Since Zencoder will be sending a POST request and won't have the authenticity token we should skip checking it.
1.  Remove the controller and action parameters from the params hash since they aren't needed, this will leave just the JSON from Zencoder.
1.  We had to unescape the string prior to parsing it with JSON.
1.  After parsing it you should be able to pull the output id and state. If the state is finished and a video was found via the output id then flag it as processed using the method we created on the video model.

## Create CarrierWave uploader

One of the things I really like about CarrierWave is that it pushes all the attachment processing code off into it's own reusable class called an uploader. Next up is creating an uploader to handle videos.

1. Generate an uploader
{% highlight bash %}
$ bundle exec rails g uploader video
{% endhighlight %}

{:start="2"}
1. Update the generated uploader, make sure you remove the **storage :file** line, we configured this in the CarrierWave initalizer
1. Update the extension whitelist to include video formats you accept
1. Uploaders have [callbacks](https://github.com/jnicklas/carrierwave/wiki/How-to%3A-use-callbacks), similar to ActiveRecord models. This is where we'll tell Zencoder the file has been uploaded.

## Creating a Zencoder job
1. Input should be the location of the video that was uploaded
1. Outputs should be an array. You can tell Zencoder to encode multiple formats, just label each hash of options appropriately (web, mobile, etc.)
1. After a job is submitted Zencoder will respond with an array of jobs that it has created. We need to loop over the array of jobs from Zencoder and grab the output id and update the model.
1. The notifications option is where we'll tell Zencoder to we want to receive the callback. This is the controller we created earlier. *In order to receive the callback Zencoder must be able to connect to your server, so it needs to be on the open internet, so edit (/config/application.rb)*

{% highlight ruby %}
config.action_mailer.default_url_options = {:host => "your_ip_address"}
{% endhighlight %}

{:start="5"}
1. Include the Rails.application.routs.url_helpers module, since routes aren't available in uploaders.
1. Set the default url to be the host of your callback, if you want to just use the same host as your email just add: Rails.application.routes.default_url_options = ActionMailer::Base.default_url_options
1. Use the ssl protocol in the callback url if your site is running ssl.
1. Edit (*/app/uploaders/video_uploader.rb*)

{% highlight ruby %}
class VideoUploader < CarrierWave::Uploader::Base
  include Rails.application.routes.url_helpers

  Rails.application.routes.default_url_options = ActionMailer::Base.default_url_options

  after :store, :zencode

  def store_dir
    "uploads/#{model.class.to_s.underscore}/#{mounted_as}/#{model.id}"
  end

  def extension_white_list
    %w(mov avi mp4 mkv wmv mpg)
  end

  def filename
    "video.mp4" if original_filename
  end

  private

  def zencode(args)
    input = "cf://rackspace_username:rackspace_api_key@blog.uploads/uploads/video/attachment/#{@model.id}/video.mp4"
    base_url = "cf://rackspace_username:rackspace_api_key@blog.uploads/uploads/video/attachment/#{@model.id}"

    zencoder_response = Zencoder::Job.create({
      :input => input,
      :output => [{
        :base_url => base_url,
        :filename => "video.mp4",
        :label => "web",
        :notifications => [zencoder_callback_url(:protocol => 'http')],
        :video_codec => "h264",
        :audio_codec => "aac",
        :quality => 3,
        :width => 854,
        :height => 480,
        :format => "mp4",
        :aspect_mode => "preserve",
        :public => 1
      }]
    })

    zencoder_response.body["outputs"].each do |output|
      if output["label"] == "web"
        @model.zencoder_output_id = output["id"]
        @model.processed = false
        @model.save(:validate => false)
      end
    end
  end

end
{% endhighlight %}


{:start="9"}
1. Mount the uploader on the video model to the attachment column. Put this line somewhere in your model (*/app/models/video.rb*).

{% highlight ruby %}
mount_uploader :attachment, VideoUploader
{% endhighlight %}

## Create Videos controller

We can now create a controller to tie it all together

1. Generate controller
{% highlight bash %}
$ bundle exec rails g controller videos index show new
{% endhighlight %}

{:start="2"}
1. Update your routes (*/config/routes.rb*):

{% highlight ruby %}
resources :videos
root :to => "videos#index"
{% endhighlight %}

{:start="3"}
1. Your controller (*/app/controllers/videos_controller.rb*) is pretty basic, the create action just needs to return success or failure on uploading the video. The Uploadify script has success and failure functions that we'll use.

{% highlight ruby %}
class VideosController < ApplicationController

  def index
  end

  def show
    @video = Video.find(params[:id])
  end

  def new
    @video = Video.new
  end

  def create
    @video = Video.new(:attachment => params[:attachment])
    if @video.save
      render :nothing => true
    else
      render :nothing => true, :status => 400
    end
  end

end
{% endhighlight %}

{:start="4"}
1. Edit your template (*/app/views/videos/index.html.erb*)
{% highlight erb %}
<%= link_to "New Video", new_video_path %>

<% Video.all.each do |video| %>
  <%= link_to "Video ##{video.id}", video_path(video) %>
<% end %>
{% endhighlight %}

{:start="5"}
1. The show view (*/app/views/videos/show.html.erb*) will either show a processing message or the video.
{% highlight erb %}
<% if @video.attachment_url.present? && @video.processed? %>
  <video width="380" height="240" controls="controls" preload="preload"><source src="<%= @video.attachment.url %>" type="video/mp4; codecs=&quot;avc1.42E01E, mp4a.40.2&quot;"></video>
<% else %>
  Please be patient while we process this video
<% end %>
{% endhighlight %}

*For a more full featured video player with Flash based fallback see [http://videojs.com/](http://videojs.com/)*

{:start="6"}
1.  Your new (*/app/videos/new.html.erb*) view doesn't need a form, just an element in the DOM to attach the flash uploader to. Use the asset_path helper for the uploadify.swf and cancel.png paths, this is used to correctly reference the files in the asset pipeline.

{% highlight html %}
<%= link_to "Video Listing", videos_path %>

<script type="text/javascript">
  $(function() {
    <% session_key = Rails.application.config.session_options[:key] %>
    var uploadify_script_data = {};
    var csrf_token = $('meta[name=csrf-token]').attr('content');
    var csrf_param = $('meta[name=csrf-param]').attr('content');

    uploadify_script_data[csrf_param] = encodeURI(encodeURIComponent(csrf_token));
    uploadify_script_data['<%= session_key %>'] = '<%= cookies[session_key] %>';

    $('#video_attachment').uploadify({
      uploader  : '<%= asset_path("jquery.uploadify-v2.1.4/uploadify.swf") %>',
      script    : '<%= videos_path %>',
      wmode     : 'transparent',
      cancelImg : '<%= asset_path("jquery.uploadify-v2.1.4/cancel.png") %>',
      fileDataName    : 'attachment',
      scriptData : uploadify_script_data,
      auto      : true,
      buttonText : 'Browse',
      onAllComplete : function(event, data){
        alert("Success!  Please be patient while the video processes.");
      },
      onError: function(event, ID, fileObj, errorObj){
        alert("There was an error with the file you tried uploading.\n Please verify that it is the correct type.")
      }
    });
  });
</script>
{% endhighlight %}

## Try it out!

Start your server and try uploading a video!

The full source code for this is located at:


[https://github.com/nick-desteffen/carrierwave_zencoder_example](https://github.com/nick-desteffen/carrierwave_zencoder_example)

*(The video format in this post only works in Chrome with the HTML5 Video tag used)*
Good luck!

## Related Links

*   [CarrierWave](https://github.com/jnicklas/carrierwave)
*   [Rackspace Cloud Files](http://www.rackspace.com/cloud/cloud_hosting_products/files/)
*   [Zencoder](http://zencoder.com/)
*   [Fog](https://github.com/geemus/fog)
*   [Amazon S3](http://aws.amazon.com/s3/)
*   [Google Storage for Developers](http://code.google.com/apis/storage/docs/signup.html)
*   [Uploadify](http://www.uploadify.com/)
*   [VideoJS](http://videojs.com/)
