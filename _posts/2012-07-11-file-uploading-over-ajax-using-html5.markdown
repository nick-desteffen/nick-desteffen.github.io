Using the HTML5 [File API](http://www.w3.org/TR/FileAPI/) to upload files over AJAX is a great way to to add file upload capabilities and not resort to using a flash based solution or iframe hacks.

It's easy to use and works with most modern browsers. This example use the [Data URI Scheme](http://en.wikipedia.org/wiki/Data_URI_scheme) to transfer the file over AJAX.

The mime type and base64 encoded file are sent as a standard parameter on a POST request. The server then decodes the image and saves it.

## Data URI Scheme

An image can be embedded in a HTML or CSS document using this method. The example below is an encoded red dot image. The browser is just reading the embedded image as it's **src** attribute _(scroll to see the entire URI)_.

<div style="overflow-y: scroll; width: 650px; background-color: #efefef;">
  data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg==
</div>

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg==)

## Client Side

On the client side your going to need to bind a change event to the file attachment field. The change event loops over all the files chosen, creates a **FileReader** object which uses the **readAsDataURL** method to encode the file using the Data URI Scheme. An object containing the orignial filename and data is then added to an array.

When the form is submitted each file in the array is read and sent across in an AJAX call.

*Client side javascript:*

{% highlight javascript %}
var files = [];

$("input[type=file]").change(function(event) {
  $.each(event.target.files, function(index, file) {
    var reader = new FileReader();
    reader.onload = function(event) {
      object = {};
      object.filename = file.name;
      object.data = event.target.result;
      files.push(object);
    };
    reader.readAsDataURL(file);
  });
});

$("form").submit(function(form) {
  $.each(files, function(index, file) {
    $.ajax({url: "/ajax-upload",
          type: 'POST',
          data: {filename: file.filename, data: file.data},
          success: function(data, status, xhr) {}
    });
  });
  files = [];
  form.preventDefault();
});
{% endhighlight %}

## Server Side

The AJAX endpoint on the server just has to read the data and reassemble the file. Since we only want the actual file data we need to cut away the mime type and encoding. So we just look for **_base64,_** and take everything after it.

*Sinatra endpoint:*

{% highlight ruby %}
post "/ajax-upload" do
  data = params[:data]
  filename = params[:filename]

  ## Decode the image
  data_index = data.index('base64') + 7
  filedata = data.slice(data_index, data.length)
  decoded_image = Base64.decode64(filedata)

  ## Write the file to the system
  file = File.new("public/uploads/#{filename}", "w+")
  file.write(decoded_image)

  "/uploads/#{filename}"
end
{% endhighlight %}

## Example Application

If you wanna just checkout a complete working example head on over to Github, clone the repo and run:

{% highlight shell %}
$ bundle install && bundle exec thin start
{% endhighlight %}

[https://github.com/nick-desteffen/html5_ajax_file_upload_demo](https://github.com/nick-desteffen/html5_ajax_file_upload_demo)

## Related Links

*   [Data URI Scheme](http://en.wikipedia.org/wiki/Data_URI_scheme)
*   [HTML5 File API](http://www.w3.org/TR/FileAPI/)
*   [HTML5 File APIs Tutorial](http://www.html5rocks.com/en/tutorials/file/dndfiles/ )
*   [Sinatra](http://www.sinatrarb.com/)
*   [Demo Application](https://github.com/nick-desteffen/html5_ajax_file_upload_demo)
*   [FileAPI polyfill for IE9 Support](https://github.com/Jahdrien/FileReader)
