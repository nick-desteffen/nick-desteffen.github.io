This post will outline the process of building a backend Ruby API wrapper using [Typhoeus](https://github.com/typhoeus/typhoeus) and [Virtus](https://github.com/solnic/virtus).

As a project gets larger and more complex it makes sense to migrate to a [service oriented architecture (SOA)](http://en.wikipedia.org/wiki/Service-oriented_architecture). This allows you to split out your application into smaller more manageable components. Deployment is simpler however it becomes more difficult to work with your data. Javascript front end frameworks like [Backbone.js](http://backbonejs.org/) and [Ember.js](http://emberjs.com/) are designed to consume data from [REST](http://en.wikipedia.org/wiki/Representational_state_transfer) APIs easily, this is great if you are building a Javascript application. If you need to connect via Ruby you are going to want an easy way of working with your data. By following some simple conventions you can build a mixin to add to your client side objects and you'll be set.

## Virtus Objects

[Virtus](https://github.com/solnic/virtus) is a gem that was extracted from the [DataMapper](http://datamapper.org/) gem. It provides the ability to define attributes and their type on a class. Instantiating a new object will cause it's attributes to be automatically coerced into your definition. You can define your attributes as simple types such as Strings and Integers, or even more complex object types such as Arrays which contain a specific type of object.

{% highlight ruby %}
class User
  include Virtus.model

  attribute :id,        Integer
  attribute :name,      String
  attribute :email,     String
  attribute :age,       Integer
  attribute :addresses, Array[Address]
end

class Address
  include Virtus.model

  attribute :id,       Integer
  attribute :street,   String
  attribute :city,     String
  attribute :state,    String
  attribute :zip_code, String
end
{% endhighlight %}

The User object above contains an array of Address objects. Passing in a hash of data to the constructor will build out the object automatically.

{% highlight ruby %}
data = {
  id: 1,
  name: "Nick",
  email: "nick@gmail.com",
  age: 33,
  addresses: [{
    id: 1,
    street: "1234 Main St.",
    city: "Chicago",
    state: "IL",
    zip_code: "60640"
  }, {
    id: 2,
    street: "555 Park Ave.",
    city: "New York",
    state: "NY",
    zip_code: "10103"
  }]
}
{% endhighlight %}

{% highlight ruby %}
2.0.0p247 :001 > user = User.new(data)
=> #<User:0x007f9a7a496418 @id=1, @email="nick@gmail.com", @name="Nick", @age=1, @addresses=[#<Address:0x007f9a7a4956f8 @id=1, @city="Chicago", @state="IL", @zip_code="60640", @street="1234 Main St.">, #<Address:0x007f9a79baa098 @id=2, @city="New York", @state="NY", @zip_code="10103", @street1=nil, @street2=nil>]>
2.0.0p247 :002 > user.addresses
=> [#<Address:0x007f9a7a4956f8 @id=1, @city="Chicago", @state="IL", @zip_code="60640", @street="1234 Main St.">, #<Address:0x007f9a79baa098 @id=2, @city="New York", @state="NY", @zip_code="10103", @street="555 Park Ave.">]
{% endhighlight %}


## Building the API

Now that you have your client objects build out, how do you build out the data from your API? I prefer to use [ActiveModel::Serializers](https://github.com/rails-api/active_model_serializers) to build out the structure. The DSL provided by the gem is very similar to ActiveRecord. You just define what attributes you want to be returned when a model is serialized. When you include an association it's own serializer is automatically brought in to build out the JSON.

{% highlight ruby %}
class UserSerializer < ActiveModel::Serializer

  attributes :id, :name, :email, :age

  has_many :addresses

end

class AddressSerializer < ActiveModel::Serializer

  attributes :id, :street, :city, :state, :zip_code

end
{% endhighlight %}

{% highlight ruby %}
class UsersController < ApplicationController

  respond_to :json

  def show
    user = User.find(params[:id])
    respond_with(user, root: false)
  end

end
{% endhighlight %}


The serializers defined above are used to build out the same structure our Virtus objects need. If the API sends additional attributes they are simply ignored by Virtus. The **respond_with** method will lookup the serializer to use based off the type of object being returned, overriding it isn't difficult to do. If you aren't a fan of ActiveModel Serializers you can take a look at [Jbuilder](https://github.com/rails/jbuilder). It provides an alternative syntax for building out JSON responses.</div>

## Getting data from the API

Now that the API is setup and is returning data how do we get it? There are a number of HTTP client gems out there such as [Typhoeus](https://github.com/typhoeus/typhoeus), [Faraday](https://github.com/lostisland/faraday) and [httparty](https://github.com/jnunemaker/httparty). The Ruby Standard Library also includes [Net::HTTP](http://ruby-doc.org/stdlib-2.0.0/libdoc/net/http/rdoc/Net/HTTP.html), however it isn't very straightforward to use. I prefer Typhoeus, it is easy to use, has a great interface and is quite flexible. All HTTP verbs work in the same manner with some minor tweaks. Getting data from the API endpoint defined above **(users#show)** can be done with 1 line.


{% highlight ruby %}
response = Typhoeus.get("http://localhost:3001/users/1", headers: {"Accept" => "application/json"})
data     = JSON.parse(response.body)
{% endhighlight %}

We can now feed the data from the response into a new Virtus object and read attributes from it like a standard ActiveRecord model.

{% highlight ruby %}
2.0.0p247 :001 > user = User.new(data)
2.0.0p247 :001 > user.name
=> "Nick"
{% endhighlight %}

## Basic CRUD Wrapper

Reading your data is great, however to be really useful to be able to make updates in addition to creating new records and removing old records. Typhoeus works with POST requests almost exactly the same as GET requests. Consistency is one the things I like about it.

{% highlight ruby %}
user_attributes = {
  email: 'dog@example.com',
  name:  'Payton',
  age:    12
}
response = Typhoeus::Request.post("http://localhost:3001/users", body: {user: user_attributes}, headers: {"content-type" => "application/x-www-form-urlencoded"})
data = JSON.parse(response.body)
user = User.new(data)
{% endhighlight %}

After parsing the response and passing it into a new object we can see that the database successfully created a new record.

{% highlight ruby %}
2.0.0p247 :001 > user.name
=> "Payton"
2.0.0p247 :002 > user.id
=> 2
{% endhighlight %}

To make things a bit more "Rails" like we can create add a few class methods to the User class.

{% highlight ruby %}
def self.find(id)
  response = Typhoeus.get("http://localhost:3001/#{id}", headers: {"Accept" => "application/json"})
  data = JSON.parse(response.body)
  return User.new(data)
end

def self.create(attributes={})
  response = Typhoeus::Request.post("http://localhost:3001/users", body: {user: attributes}, headers: {"content-type" => "application/x-www-form-urlencoded"})
  data = JSON.parse(response.body)
  return User.new(data)
end

def self.update(id, attributes={})
  user = User.new(attributes.merge(id: id))
  response = Typhoeus::Request.put("http://localhost:3001/users/#{id}", body: {user: attributes}, headers: {"content-type" => "application/x-www-form-urlencoded"})
  return user
end

def self.destroy(id)
  response = Typhoeus::Request.delete("http://localhost:3001/users/#{id}")
  return response.success?
end
{% endhighlight %}

{% highlight ruby %}
2.0.0p247 :001 > User.find(2)
=> #<User:0x007fecfacb2e08 @id=2, @name="Payton", @email="dog@example.com", @age=12>
2.0.0p247 :002 > User.create(name: "Jack", email: "jack@example.com", age: 8)
=> #<User:0x007fecfcc97930 @id=3, @name="Jack", @email="jack@example.com", @age=8>
{% endhighlight %}

The methods aren't a 100% match to Rails **update_attributes** and **destroy** however it is still an external call, so we want to minimize the number of requests. Instead of finding the user, then updating it we are doing it in one call.</div>

## Adding ActiveModel

Now that we have a User class that handles all the basic CRUD operations we can toss in a bit of ActiveModel to make it work with standard Rails forms. The **respond_with** method that returns JSON from the API will also return any validation errors. By mixing in **ActiveModel::Validations** we get access to the errors object. If the response status is **422** we can take it and build out the errors on our local object.

{% highlight ruby %}
class User
  include Virtus.model
  extend  ActiveModel::Naming
  extend  ActiveModel::Translation
  include ActiveModel::Conversion
  include ActiveModel::Validations

  attribute :id,    Integer
  attribute :name,  String
  attribute :email, String
  attribute :age,   Integer

  def persisted?
    id.present?
  end

  def self.create(attributes={})
    response = Typhoeus::Request.post("http://localhost:3001/users", body: {user: attributes}, headers: {"content-type" => "application/x-www-form-urlencoded"})
    if response.success?
      data = JSON.parse(response.body)
      user = User.new(data)
    else
      user = User.new(attributes)
      if response.response_code == 422
        data = JSON.parse(response.body)
        user.assign_errors(data)
      end
    end
    return user
  end

  def assign_errors(error_data)
    error_data["errors"].each do |attribute, attribute_errors|
      attribute_errors.each do |error|
        self.errors.add(attribute, error)
      end
    end
  end

end
{% endhighlight %}


{% highlight ruby %}
2.0.0p247 :001 > user = User.create(email: "")
=> #<User:0x007f8f7a4d64b0 @email="", @id=nil, @name=nil, @age=nil, @addresses=[], @errors=#<ActiveModel::Errors:0x007f8f7b4cd490 @base=#<User:0x007f8f7a4d64b0 ...>, @messages={:name=>["can't be blank"], :email=>["can't be blank"]}>>
2.0.0p247 :002 > user.errors[:email]
=> ["can't be blank"]
{% endhighlight %}

Since we are using the same interface ActiveRecord uses, our forms will behave just like everything was connected directly to our database.</div>

## Abstracting for Reuse

So now we have a user class that works like ActiveRecord, but goes through a REST API instead. If your application has a bunch of classes that connect to the same API things are going to get very repetitive. You can create a mixin to include on all your objects. The one below has everything used in this post, plus basic search and listing **(where / all)**.

{% highlight ruby %}
module ApiBase
  extend ActiveSupport::Concern

  included do
    require "addressable/uri"
    include Virtus.model
    extend  ActiveModel::Naming
    extend  ActiveModel::Translation
    include ActiveModel::Conversion
    include ActiveModel::Validations
  end

  def persisted?
    id.present?
  end

  def assign_errors(error_data)
    error_data[:errors].each do |attribute, attribute_errors|
      attribute_errors.each do |error|
        self.errors.add(attribute, error)
      end
    end
  end

  module ClassMethods

    def find(id)
      response = Typhoeus.get("#{base_url}/#{id}", headers: {"Accept" => "application/json"})
      if response.success?
        data = JSON.parse(response.body, symbolize_names: true)
        return self.new(data)
      else
        return nil
      end
    end

    def where(parameters={})
      parameters.reject!{ |key, value| value.blank? }
      querystring = Addressable::URI.new.tap do |uri|
        uri.query_values = parameters
      end.query

      response = Typhoeus.get("#{base_url}?#{querystring}", headers: {"Accept" => "application/json"})
      if response.success?
        data = JSON.parse(response.body, symbolize_names: true)
        return data.map{ |record| self.new(record) }
      else
        return nil
      end
    end

    alias_method :all, :where

    def create(attributes={})
      response = Typhoeus::Request.post(base_url, body: envelope(attributes), headers: {"content-type" => "application/x-www-form-urlencoded"})
      data = JSON.parse(response.body, symbolize_names: true)
      if response.success?
        object = self.new(data)
      else
        object = self.new(attributes)
        object.assign_errors(data) if response.response_code == 422
      end
      return object
    end

    def update(id, attributes={})
      object = self.new(attributes.merge(id: id))
      response = Typhoeus::Request.put("#{base_url}/#{id}", body: envelope(attributes), headers: {"content-type" => "application/x-www-form-urlencoded"})
      if response.response_code == 422
        data = JSON.parse(response.body, symbolize_names: true)
        object.assign_errors(data)
      end
      return object
    end

    def destroy(id)
      response = Typhoeus::Request.delete("#{base_url}/#{id}")
      return response.success?
    end

  private

    def envelope(attributes)
      envelope = {}
      envelope[self.name.to_s.underscore.downcase] = attributes
      return envelope
    end

    def base_url
      Rails.configuration.api_endpoint + "/" + self.name.to_s.underscore.downcase.pluralize
    end

  end

end
{% endhighlight %}

The User class now has the all functionality it had before, yet the mixin can be added to make any other class just as functional.

{% highlight ruby %}
class User
  include ApiBase

  attribute :id,        Integer
  attribute :name,      String
  attribute :email,     String
  attribute :addresses, Array[Address]
end
{% endhighlight %}

## Using Hashie as an alternative

If you don't need full CRUD or a lot of complexity you can also just use [Hashie](https://github.com/intridea/hashie). It will take a hash and construct a simple object with accessors.

{% highlight ruby %}
2.0.0p247 :01 > data = JSON.parse(Typhoeus.get("http://localhost:3001/users/1", headers: {"Accept" => "application/json"}).body)
=> {"id"=>3, "name"=>"Nick", "email"=>"nick@gmail.com", "age"=>"33", "addresses"=>[{"id"=>1, "street"=>"1234 Main St.", "city"=>"Chicago", "state"=>"IL", "zip_code"=>"60640"}, {"id"=>2, "street"=>"555 Park Ave.", "city"=>"New York", "state"=>"NY", "zip_code"=>"10103"}]}
2.0.0p247 :02 > user = Hashie::Mash.new(data)
=> #<Hashie::Mash addresses=[#<Hashie::Mash city="Chicago" id=1 state="Il" street="1234 Main St." zip_code="60640">, #<Hashie::Mash city="New York" id=2 state="NY" street="555 Park Ave." zip_code="10103">] email="nick@gmail.com" name="Nick" id=1 age=33>
2.0.0p247 :03 > user.email
=> "nick@gmail.com"
{% endhighlight %}


## Demo Applications

[https://github.com/nick-desteffen/virtus-typhoeus-api-demo](https://github.com/nick-desteffen/virtus-typhoeus-api-demo)
I made a couple of demo applications. To get up and running just clone the repo and fire up both applications.

To start the API:

{% highlight shell %}
$ cd api
$ bundle install
$ bundle exec rails s -p 3001
{% endhighlight %}

To start the client:

{% highlight shell %}
$ cd client
$ bundle install
$ bundle exec rails s
{% endhighlight %}

Now visit: [http://localhost:3000](http://localhost:3000)

The API wrapper mixin is located at:
[https://github.com/nick-desteffen/virtus-typhoeus-api-demo/blob/master/client/lib/api/base.rb](https://github.com/nick-desteffen/virtus-typhoeus-api-demo/blob/master/client/lib/api/base.rb)

## Related Links

*   [Typhoeus](https://github.com/typhoeus/typhoeus)
*   [Virtus](https://github.com/solnic/virtus)
*   [Hashie](https://github.com/intridea/hashie)
*   [Datamapper](http://datamapper.org)
*   [ActiveModel::Serializers](https://github.com/rails-api/active_model_serializers)
*   [Demo Application Repo](https://github.com/nick-desteffen/virtus-typhoeus-api-demo)
*   [Faraday](https://github.com/lostisland/faraday)
*   [httparty](https://github.com/jnunemaker/httparty)
*   [Net::HTTP](http://ruby-doc.org/stdlib-2.0.0/libdoc/net/http/rdoc/Net/HTTP.html)
*   [Jbuilder](http://ruby-doc.org/stdlib-2.0.0/libdoc/net/http/rdoc/Net/HTTP.html)
*   [SOA Wikipedia Entry](http://en.wikipedia.org/wiki/Service-oriented_architecture)
*   [REST Wikipedia Entry](http://en.wikipedia.org/wiki/Representational_state_transfer)
