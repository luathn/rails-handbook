Adapters are objects whose purpose is to wrap third-party API calls. The idea behind adapter objects is to introduce an abstraction layer
over our external API calls. This makes all the requests going from our application to outside third-party APIs centralized in one place.
Also, if we want to replace the existing gem for API calls for some reason, we can easily do so without the need to search the entire codebase and replace each method.

Adapter objects are placed inside the /app/adapters/ folder.

## Bad solution

Having API calls in controller or service objects. We are instantiating an Instagram client from the controller and keeping the token as a constant in the controller. If we have similar code scattered around as well, it will be difficult to carry out changes if we want to replace the gem, or if the API of the gem itself is changed.

```ruby
class PhotosController < ApplicationController
  Rails.application.secrets.instagram_access_token

  def index
    instagram_client = Instagram.client(access_token: INSTAGRAM_ACCESS_TOKEN)
    feed = instagram_client.user_recent_media

    render json: feed
  end
end
```

## Better solution

We extract all external API calls to a single, InstagramAdapter. Also, all the data necessary to instantiate the client is moved to the adapter class. This way, it is easy to swap the gem for some other. Also, we are not breaking the Single Responsibility Principle, since the InstagramAdapter class handles calls to the Instagram API.

If we are making a couple of calls to Facebook, Instagram, GitHub, etc., this solution is going to be sufficient; especially if we already rely on an existing gem (in this case, we are using the Instagram gem, and InstagramAdapter is just a wrapper).

```ruby
class PhotosController < ApplicationController
  def index
    render json: instagram_client.recent_media_with_location
  end

  def instagram_client
    InstagramAdapter.new
  end
end
```

```ruby
class InstagramAdapter
  INSTAGRAM_ACCESS_TOKEN = "9999"

  def initialize
    @client = Instagram.client(access_token: INSTAGRAM_ACCESS_TOKEN)
  end

  def recent_media
    @recent_media ||= @client.user_recent_media(count: 50)
  end

  def recent_media_with_location
    recent_media.reject { |item| item.location.nil? }
  end
end
```

In this case, there is a single ```instagram_adapter.rb``` class located in ```/app/adapters/instagram/```. If the single class starts to get too large, we can separate the concerns based on API calls. For example, let's say we have a GitHub adapter. We can have two adapters—one for commits, and one for repos. So the structure would be as follows ```app/adapters/github/commits_adapter.rb``` and ```app/adapters/github/repos_adapter.rb```, and we would have a base class for connecting to the API located in ```app/adapters/github/base_adapter.rb```. Except for multiple files, there is not much difference from the example above.

## Going beyond simple adapters

If we are going to build a complete wrapper around a third-party API (e.g., Dolcela, PTV, etc.), or if we need options for manipulating requests and responses from the API (e.g., GitHub, Instagram, Facebook), using only adapters won't be sufficient.

This is especially the case if the external API is using XML and we are using JSON in our responses, or if we don't have a complete gem we can rely on and need to develop our own custom solution.

So, to tackle these kinds of problems, we are going to add serializer and deserializer objects.
Serializers—used for preparing the request before it goes to external API.
Deserializers—used for parsing responses from the external API.

## Deserializer objects

Deserializers are used when we want to parse the response we received from the external API. This is very frequent when the response format from the API does not quite meet our application's needs.

Deserializers are stored in ```/app/adapters/{api_service}/deserializers/```, where ```api_service``` is the external API, e.g., Instagram, Dolcela, PTV, etc.

In the following example, we instantiate the deserializer object by sending the API response to the initialize method.

**Example of a deserializer object**

```ruby
module Instagram
  module Deserializers
    class RecentMedia
      attr_reader :response_body

      def initialize(response_body)
        @response_body = response_body
      end

      def success?
        stripped_response_body[:response_code] == "0"
      end

      def failed?
        !success?
      end

      def status_code
        stripped_response_body[:response_code]
      end

      def status_message
        stripped_response_body[:response_code_description]
      end

      private

      def stripped_response_body
        @stripped_response_body ||=
          @response_body[:hosted_page_authorize_response][:hosted_page_authorize_result]
      end
    end
  end
end
```

**Using a deserializer in an adapter**

In the above example, after we'd fetched the recent user media from Instagram, we passed the response to the RecentMedia parser which is going to parse the response and add some additional methods around the response (for example: failed?, success?, etc.).

```ruby
module Adapters
  class InstagramAdapter
    INSTAGRAM_ACCESS_TOKEN = "9999"

    def initialize
      @client = Instagram.client(access_token: INSTAGRAM_ACCESS_TOKEN)
    end

    def recent_media
      Instagram::Deserializer::RecentMedia.new(raw_recent_media)
    end

    def recent_media_with_location
      recent_media.reject { |item| item.location.nil? }
    end

    private

    def raw_recent_media
      client.user_recent_media(count: 50)
    end
  end
end
```

**Deserializing collections**

If you want to deserialize a collection you received from the API, you can make collection objects. In this case, the itemize method makes a collection of MediaItem objects.

```ruby
module Instagram
  class RecentMedia

    def initialize(instagram_user_id)
      @instagram_user_id = instagram_user_id
    end

    def instagram_recent_media
      Instagram.user_recent_media(instagram_user_id, count: 50)
    end

    def feed
      @feed ||= instagram_recent_media.map { |item| Deserializer::MediaItem.new(item) }
    end

    def items_with_location_present
      feed.reject { |item| item.restaurant_name.nil? }
    end

    def items_with_location_present_in_json
      items_with_location_present.map(&:to_json)
    end
  end
end

class InstagramAdapter
  INSTAGRAM_ACCESS_TOKEN = "9999"

  def initialize
    @client = Instagram.client(access_token: INSTAGRAM_ACCESS_TOKEN)
  end

  def user_recent_media
    itemize(@client.user_recent_media)
  end

  private

  def itemize(user_recent_items)
    user_recent_items.map { |item| MediaItem.new(item) }
  end
end
```

```ruby
module Instagram
  class MediaItem
    attr_reader :item

    def initialize(item)
      @item = item
    end

    def restaurant_name
      item.location.name if item.location?
    end

    def image_url
      item.images.standard_resolution.url
    end
  end
end
```

## Serializer objects

Serializers are used to prepare the request before we send it to the server. This is especially useful if we need to make an XML request.

```ruby
module Dolcela
  class RecipeSerializer
    def initialize(id)
      @id = id
    end

    def method
      "post"
    end

    def request_body
      @xml_request_body ||= "<?xml version='1.0'?>" + Gyoku.xml(
        method_call: {
          method_name: "recipes.get",
          params: {
            param: {
              value: {
                struct: {
                  member: [
                    {
                      name: "content_id",
                      value: { int: @id }
                    },
                    {
                      name: "omit_author_data",
                      value: { int: 1 }
                    },
                    {
                      name: "get_basic_data",
                      value: { boolean: 1 }
                    },
                    {
                      name: "include_preparation_steps",
                      value: { boolean: 1 }
                    },
                    {
                      name: "get_action_shots",
                      value: { boolean: "1" }
                    },
                    {
                      name: "include_ingredients",
                      value: { boolean: 1 }
                    }
                  ]
                }
              }
            }
          }
        }
      )
    end
  end
end
```

We are going to use serializers in the adapter objects, for example:

```ruby
module Coolinarika
  module Adapter
    class RecipeAdapter < BaseAdapter
      def recipe(id)
        @id = id
        RecipeDeserializer.new(fetch_recipe)
      end

      private

      def fetch_recipe
        execute_request RecipeSerializer.new(@id)
      end
    end
  end
end
```

Here is an example of what the BaseAdapter class looks like. It is a base class for all adapters and contains the execute_request method for making requests to the API.

```ruby
module Coolinarika
  module Adapter
    class BaseAdapter

      API_BASE_URL = "http://www.coolinarika.com/api/"

      def self.execute_request(request)
        RestClient::Request.execute(request.method,
                                    url: API_BASE_URL,
                                    payload: request.body,
                                    headers: { content_type: "application/xml" }
                                   )
      end
    end
  end
end
```

## Benefits

Creating an adapter object allows you to provide a layer of abstraction around your external libraries. Since you decide what interface your adapter is going to expose, it’s easy to use another library that does the same job. In such cases, you only have to change the adapter’s code.

If you have a code that you cannot change, and it has a dependency which you provide, you can use an adapter to easily exchange this dependency for something else. This is especially useful if you have code which uses a legacy gem, and you want to get rid of it and provide a new gem with the same functionality (but different API).

Adapters can also be useful for testing. You can easily exchange a real integration with an external service (like Facebook) for an object which returns prepared responses. This calls an in-memory adapter, and it’s a very useful technique to make your tests run faster.

## More info

* [Arkency Ruby Rails Adapters](http://blog.arkency.com/2014/08/ruby-rails-adapters/)
* [Adapter Design Pattern in Rails Application](http://rustamagasanov.com/blog/2014/11/16/adapter-design-pattern-usage-in-rails-application-on-examples/)
