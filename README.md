# Api::Presenter

This gem builds the basics for presenting your data using the media type described in the [api doc](https://github.com/ncuesta/api-doc). Here you will find classes to represent your resources
and also the functions to convert them to json.

## Installation

Add this line to your application's Gemfile:

    gem 'api-presenter'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install api-presenter

## Usage

So, first you should now that there are three kinds of resources:

* A simple resource (Api::Presenter::Resource) that represents a single unit of information.
* A collection resource (Api::Presenter::CollectionResource) that represents an homogeneous group of resources.
* A search resource (Api::Presenter::SearchResource) it's like a collection resource but adds the query information.

### Simple Resource

Now, most probably you are needing to represent some class that contains your model information using the media type
described by [api-doc](https://github.com/ncuesta/api-doc). For this you will be needing two things:

1. Create a class that represents your resource.
2. Let your model class know how to turn into a resource.

Let's say you have a Person model:

```ruby
class Person
  attr_accessor :name, :age
  
  def initialize(name, age)
    @name = name
    @age = age
  end
end
```

In order to be able to represent this resource we need to have a PersonResource class

```ruby
class PersonResource < Api::Presenter::Resource

  property :name
  property :age
  
  def self_link
    "/person/#{@resource.name}"
  end
end
```

We should use the property method to define each of our resource properties.

The self_link definition tell which is the link that represents itself.

So now there is only one more thing left to do: Add the to_resource method used to convert the model
into a resource.

```ruby
class Person
  def to_resource
    PersonResource.new(self)
  end
end
```

And that's it, now we can get the representation in json using.

```ruby
data = Person.new("Alvaro", 27)

resource = data.to_resource # or just PersonResource.new(data)

resource.present # or Api::Presenter::Hypermedia.present resource
```

It will look like this:
```json
{
  "links":
  {
    "self":
    {
      "href": "/person/Alvaro"
    }
  }
  "name": "Alvaro",
  "age": 27
}
```
### Related resources

It's very common that our model is related to others, and we may want to show this in our representation.
To do so, need to create a resource class for each one and add them to our ```resource``` array in
hypermedia_properties.

Using the example above, now our person has a dog. So:

```ruby
class DogResource < Api::Presenter::Resource
  property name
  property owner

  def self_link
    "/dog/#{@resource.name}"
  end
end

class Dog
  attr_accessor :name, :owner
  
  def initialize(name, owner)
    @name = name
    @owner = owner
  end
  
  def to_resource
    DogResource.new(self)
  end
end

class PersonResource < Api::Presenter::Resource
  property :name
  property :age
  property :dog
end
```

Finally we present it:

```ruby
person = Person.new("Alvaro", 27)

dog = Dog.new("Cleo", person)

person_resource = person.to_resource # or just PersonResource.new(person)

person_resource.present # Api::Presenter::Hypermedia.present person_resource
```

It will look like this:

```json
{
  "links":
  {
    "self":
    {
      "href": "/person/Alvaro"
    },
    "dog":
    {
      "href": "/dog/Cleo"
    }
  }
  "name": "Alvaro",
  "age": 27
}
```

### Collection resource

A collection resource it's basically any collection that contains objects that responds to ```to_resource```,
also must respond to:

* total
* offset
* limit
* each

Now, using the Person example:

```ruby
# building a collection on top of array that responds to each, total, limit and offset
class Collection
  attr_reader :col

  def initialize(col = [])
    @col = col
  end
  
  def each(&block)
    @col.each(&block)
  end
  
  def total
    @col.count
  end
  
  def limit
    10
  end
  
  def offset
    0
  end
end

family = Collection.new([Person.new("Joe", 50), Person.new("Jane", 45), Person.new("Timmy", 10), Person.new("Sussie", 12)])

class FamilyResource < Api::Presenter::CollectionResource
  def self_link
    "/family"
  end
end

Api::Presenter::Hypermedia.present FamilyResource.new(family)
```

It will look like this:

```json
{
  "links":
  {
    "self":
    {
      "href": "/family"
    },
  }
  "offset": 0,
  "limit": 10,
  "total": 4,
  "entries":
  [
    {
      "self":
      {
        "href" : "/person/Joe"
      }
    },
    {
      "self":
      {
        "href" : "/person/Jane"
      }
    },
    {
      "self":
      {
        "href" : "/person/Timmy"
      },
    },
    {
      "self":
      {
        "href" : "/person/Sussie"
      }
    }
  ]
}
```

### Search resource

This is a special case of ```Api::Presenter::CollectionResource``` where it also has a query string and parameters.
The main difference with a Collection is that it receives as a parameter, the parameters which where used to build
the collection and also adds them to the response.

The method ```self.hypermedia_query_parameters``` determins which parameters are used in the search. It's later used
by the helper method ```query_string``` to build the query string.

```ruby
class PersonSearchResource < Api::Presenter::SearchResource
  def self.hypermedia_query_parameters
    ["name", "age"]
  end

  def self_link
    "/search_person#{query_string}"
  end
end

search = PersonSearchResource.new(Collection.new([Person.new("Joe", 50), Person.new("Jane", 45)]), age: 45)

Api::Presenter::Hypermedia.present search
```

It will look like this:

```json
{
  "links":
  {
    "self":
    {
      "href": "/search_person?query[age]=45query[name]="
    },
  }
  "offset": 0,
  "limit": 10,
  "total": 2,
  "query":
  {
    "age": 45,
    "name": nil
  },
  "entries":
  [
    {
      "self":
      {
        "href" : "/person/Joe"
      }
    },
    {
      "self":
      {
        "href" : "/person/Jane"
      }
    }
  ]
}
```

### Adding additional links

When building a resource there may be the need to build custom links. In order to do so, you need to define
two methods in your resource class:

```ruby
def custom_link
  "/path/to/custom_link"
end


def custom_link?(options = {})
  a_condition_that_determins_if_it_should_be_displayed returning true or false
end
```


## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request