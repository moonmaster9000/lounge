Note: this gem is not yet released.


# Lounge

Simple, powerful CouchDB abstractions for Ruby.

## Why CouchDB?

CouchDB is a general-purpose document datastore with incremental map-reduce. It's not for everyone, and it won't solve every problem. 

As far as databases go, though, it has some really fantastic and unique properties:

* Incremental Map/Reduce
* Schema-less JSON documents
* RESTful API (there are no low level drivers; CouchDB is a web server, and it speaks REST). 
* 2-tier web-application architecture (you can serve an entire web-application directly out of CouchDB, cutting out middlemen like Rails)
* N-master Replication


## Why Lounge?

There are several competing Ruby libraries out there for CouchDB, and they all have their strengths and weaknesses. 

Here's why you should use Lounge:

* Everything is a mixin
* You don't have to define properties on your models (CouchDB is schemaless, so why force your models into a schema?)
* Design Documents are de-coupled from models
* Built-in support for CouchDB design document features like show, list, and (fulltext) search
* Built-in integration with CouchDB-Lucene
* Fantastic API
* Easy to isolate your Javascript and unit-test it


## Getting started: configuring database / server connectivity

By default, `Loung` assumes a Rails 3 app, and will detect your app's environment via `Rails.env`. If you're using this in something other than a Rails 3 app,
then simply override the default environment detection:

```ruby
# Sinatra example
Lounge::Config.edit do
  environment do
    settings.environment
  end
end

# Rack example
Lounge::Config.edit do
  environment do
    ENV['RACK_ENV'] || 'development'
  end
end
```

### Configuring the default database

Suppose you want all of your lounge document models to use the same database in your application. No problem! You can use the `database` configuration method without
any arguments:

```ruby
Lounge::Config.edit do
  database do
    default "my_db_#{Rails.env}"
  end
end
```

This means that the default database name for all of your models will be "my_db_" followed by your Rails environment.

You could customize the database names per environment further:

```ruby
Lounge::Config.edit do
  database do
    default    "my_db_#{Rails.env}"
    production "my_production_db"
    test       "funny_test_db_name"
  end
end
```

For any environment not explicitly configured, it will fall back to the database name.

### Configuring the database for a model

To set the database for a model, use the `database` method. For example, suppose we'd like to set the database name for our `Book` document model to 
`library` in all environments:

```ruby
Lounge::Config.edit do
  database Book do
    default "library"
  end
end
```

Or, perhaps we'd like to differentiate the name between production, development, and test environments:

```ruby
Lounge::Config.edit do
  database Book do
    production "library_production"
    development "library_development"
    test "library_test"
  end
end
```

In a Rails app, this could be simplified to:

```ruby
Lounge::Config.edit do
  database Book do
    default "library_#{Rails.env}"
  end
end
```

### Configuring the database for a set of models

Similarly, you could set the database for a whole set of models in one of two ways:

1. Make every model inherit from the same parent (or mixin the same module), and set the parent's database via the `database` method
2. Pass several models to the `database` method

### Inheritance / Mixins

Let's imagine that our `Book`, `Author`, and `Genre` models all mixed in the `Library` module:

```ruby
module Library; end
class Book   ; include Lounge::Document; include Library; end
class Author ; include Lounge::Document; include Library; end
class Genre  ; include Lounge::Document; include Library; end
```

To make the `Book`, `Author`, and `Genre` models use the same database, simply set the `Library` database in the config:

```ruby
Lounge::Config.edit do
  database Library do
    default "library"
  end
end
```

Now, the database for `Book`, `Author`, and `Genre` will all be set to the same database, "library".

### Passing several models to the `database` method

Suppose `Book`, `Author`, and `Genre` couldn't all share the same ancestor, yet we'd still like all of them to share the same database;
then we could simply pass all of the models to the `database` method:

```ruby
Lounge::Config.edit do
  database Book, Author, Genre do
    default "library"
  end
end
```

### Configuring the CouchDB server 

Without any configuration, Lounge::Config will assume a CouchDB server at "http://127.0.0.1:5984". 

If you'd like to set a default server for all models regardless of environment, then try:

```ruby
Lounge::Config.edit do
  server do
    default "http://admin:password@localhost:5984"
  end
end
```

If you wanted to change the server to be different in the `production` environment:

```ruby
Lounge::Config.edit do
  server do
    default    "http://admin:password@localhost:5984"
    production "https://root:blah@my.production.server:5984"
  end
end
```

Now, in the production environment, it will connect to CouchDB server `my.production.server`; in all other environments, it will connect to `localhost`.

If you'd like to change the CouchDB server for a specific model or set of models, simply set the database name for the model (or models) to the entire
CouchDB uri for the database:

```ruby
Lounge::Config.edit do
  database Blog do
    default "http://localhost:5984/blog"
  end
end
```


## Creating document models with Lounge::Document 

CouchDB databases are just collections of documents. There are two types of documents: 

* Design Documents (special CouchDB documents in which you design how you access and manipulate your data within that database)
* Data Documents 

All documents are stored as JSON.

Let's start by creating a data model. Let's imagine we want to store some articles for a website. Simple! Let's start with the following code:

```ruby
class Article
  include Lounge::Document
end
```

We can now create an article:

```ruby
article = Article.create :title => "Lounge: CouchDB abstractions for Ruby"
```

### Adding properties to your documents

Lounge documents are schemaless. You can add any number of properties and embedded documents into your article instance:

```ruby
article.author = "Matt Parker"
article.publish_date = Time.now
article.save
```

As you can see, we didn't have to define any scheme or "properties" on our `Article` model to start adding structure to it. Just like real CouchDB documents, `Lounge::Document` documents are schemaless. 

Calling the `save` method on our article will store it in the database:

```ruby
article.save
  #==> true
```

Now our article has an "_id" property (set by CouchDB). You can access it by either `_id` or `id`:

```ruby
article._id
  #==> 8903kjdsklfjdskl9e84032890432
```

or... 

```ruby
article.id
  #==> 8903kjdsklfjdskl9e84032890432
```
 

## Finding Documents

At it's most basic level, CouchDB is a key/value store. The key is the id of a document, and the value returned is the document itself. You can retrieve a document by it's ID by calling the `get` method on your model:

```ruby
Article.create :id => "lounge-ruby-library-for-couchdb"

a = Article.get "lounge-ruby-library-for-couchdb"

a.id.should == "lounge-ruby-library-for-couchdb"
```

To help ease the transition from other CouchDB libraries, and to add familiarity for those transitioning from ActiveRecord, you can alterantively use the `find` method. It's simply an alias for `get`:

```ruby
Article.find "my-article"
```

## Designing access to your database (Querying)

In CouchDB, if you want to move beyond simple key/value access of your documents, you have to design your own data access patterns. You accomplish this by creating a special "design" document.

### Lounge::Design

The `Lounge` library allows you to create design documents related to a specific model, or to decouple the design document from any specific model and create access patterns for your data across multiple models. The former is by far a more common use case.

Let's start by designing ways to access our `Article` model. Suppose we have a need to lookup articles by their `created_at` timestamp. We can use the `map` class method on the model (provided by the `Lounge::Design` mixin) to accomplish this:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  timestamps!

  map :created_at
end
```

This will generate a design document, `_design/Article`, with a view called "by_created_at" that maps `Article` documents by their `created_at` timestamps. The actual map function looks like this:

```javascript
function(doc){
  if (doc.lounge_type == 'Article')
    emit(doc.created_at, null)
}
```

If you're unfamiliar with map/reduce, you can read more about the CouchDB implementation of map/reduce here: http://wiki.apache.org/couchdb/Introduction_to_CouchDB_views#Basics

We can query this view with the `map_by_created_at` class method on our `Article` model:

```ruby
Article.map_by_created_at.startkey(1.day.ago).each {...}
```

Translated, this means "select all of the articles created in the last 24 hours".

`Lounge` automatically added a `reduce` onto your `by_created_at` view that counts the number of items in the index. Therefore, if you want to count how many results there are, use the `count_by_created_at` method.

For example, to determine how many articles were created in the last 24 hours:

```ruby
Article.count_by_created_at.startkey(1.day.ago).get!
```

The `get!` method tells the query proxy to immediately fetch the result from the database.

`Lounge` supports all of the CouchDB query options, not just the `startkey` and `endkey`. You can chain together as many of them as you want:

* limit          
* skip           
* key            
* startkey       
* endkey         
* startkey_docid 
* endkey_docid   
* stale          
* descending     
* group          
* group_level    
* reduce         
* include_docs   
* update_seq     

You can find detailed documentation on all of these query options here: http://wiki.apache.org/couchdb/HTTP_view_API

### Maps on multiple properties

You can create a compound key of multiple properties by simply passing all of the properties to the `map` method. For example, suppose we'd like to be able to find all of the articles written by a specific author created in the last 24 hours:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design
  
  timestamps!

  map :author, :created_at
end
```

This will generate a view within the `_design/Article` design document titled "by_author_and_created_at"; the map function will look like this:

```javascript
function(doc){
  if (doc.lounge_type == 'Article')
    emit([doc.author, doc.created_at], null)
}
```

You could now find all of the articles created by a specific author within the last 24 hours thusly:

```ruby
Article.map_by_author_and_created_at.startkey(["Matt Parker", 1.day.ago]).endkey(["Matt Parker", Time.now]).each {...}
```

### Map classes

There will come a time when you need to create a custom map function, when your needs go beyond simply mapping over properties. In that situation, you can create a map class. 

Let's imagine that we want to find all the articles in the system that have more than 10 comments.

```ruby
class ByCommentCount
  include Lounge::Map

  def map
    <<-JS
    function(doc){
      if (#{conditions}){
        emit(doc.comments.length, null)
      }
    }
    JS
  end
end
```

Notice that we included `Lounge::Map` into our class, and that we referenced the `conditions` method inside of our map function.

`ByCommentCount.new.map` would return the following:

```javascript
function(doc){
  if (true)
    emit(doc.comments.length, null)
}
```

Basically, because we're not using `ByCommentCount` within the context of a document, the `conditions` method simply defaults to returning `true`.

However, if we use this map class inside of our article model:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  map ByCommentCount
end
```

then `Lounge` will create a view "by_comment_count" on your `_design/Article` design document with the following map function:

```javascript
function(doc){
  if (doc['lounge_type'] == 'Article')
    emit(doc.comments.length, null)
}
```

As you can see, used within the context of the `Article` class, the `conditions` method on our `ByCommentCount` class returned `doc['lounge_type'] == 'Article'`.


### Conditions

Sometimes you'll need to filter your documents by some sort of criteria. In CouchDB, you can accomplish this by creating several views, each with a map that filters documents by the criteria you desire. 

This process can be tedious, and you'll end up with several similarly named maps. But never fear! `Lounge` hides the nasty, leaving you with a tasty API. 

Let's imagine we want to filter our map over the `created_at` property by articles originally written for print, and for articles published anonymously:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  timestamps!

  map :created_at do
    conditions do
      anonymous :author => nil
      print     :issue  => not(nil)
    end
  end
end
```

This translates just as you would expect: print articles articles have a non-nil issue property; anonymous articles have a nil author property.

`Lounge` will read this definition and create several views inside the `_design/Article` design document. You can now use `print` and `anonymous` in your query proxy, either in isolation, or simultaneously.

To fetch all articles created in the last 24 hours written originally for print:

```ruby
Article.map_by_created_at.startkey(1.day.ago).print.each {...}
```

To fetch all articles created in the last 24 hours published anonymously:

```ruby
Article.map_by_created_at.startkey(1.day.ago).anonymous.each {...}
```

To fetch all articles created in the last 24 hours published anonymously and written originally for print:

```ruby
Article.map_by_created_at.startkey(1.day.ago).anonymous.print.each {...}
```

This API will take you a long way, but sometimes your needs will require you to write the actual javascript conditional. Let's imagine that we want to filter all articles that have not been commented on ("unpopular" articles):

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  timestamps!

  map :created_at do
    conditions do
      anonymous :author => nil
      print     :issue  => not(nil)
      unpopular "doc.comments.length == 0"
    end
  end
end
```

In this case, we've written the javascript conditional directly. 

Lastly, it's often advantageous to make your conditions reusable. You might want to use them in various models. You can do so by turning them into modules:

```ruby
module Anonymous
  def conditions
    "#{super} && doc.author == null"
  end
end

module Print
  def conditions
    "#{super} && doc.issue !== null"
  end
end

module Unpopular
  def conditions
    "#{super} && doc.comments.length == 0"
  end
end
```

With these modules, you can now use them as conditions within any design class:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  timestamps!

  map :created_at do
    conditions Anonymous, Print, Unpopular
  end
end
```

`Lounge` will convert the class names into lowercased, underscored condition methods on your query proxy. In this case, you would end up with the query proxy methods `anonymous`, `print`, and `unpopular`. If you desire, you can create different names:  

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  timestamps!

  map :created_at do
    conditions do 
      filter_by_anonymous Anonymous
      filter_by_print     Print
      filter_by_unpopular Unpopular
    end
  end
end
```

In most apps, you'll likely namespace your condition modules, placing them inside some other module. `Lounge`, however, will not namespace your proxy methods. THIS IS BY DESIGN.

For example, imagine we had namespaced our `Anonymous`, `Print`, and `Unpopular` modules inside a module called `Conditions`:

```ruby
module Conditions
  module Anonymous 
    ...
```

Now if we included these modules as conditions in our `created_by` map:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  timestamps!

  map :created_at do
    conditions Conditions::Anonymous, Conditions::Print, Conditions::Unpopular
  end
end
```

we would still call query proxy methods `anonymous`, `print`, and `unpopular`:

```ruby
Article.map_by_created_at.anonymous.print.unpopular.each {...}
```

If you need to change the autogenerated name of a query proxy method, simply specify it in a block passed to your `conditions`:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  timestamps!

  map :created_at do
    conditions do
      written_anonymously Conditions::Anonymous
      written_for_print   Conditions::Print
      unpopular           Conditions::Unpopular
    end
  end
end
```

### Naming your views

It's sometimes advantageous to create a specific name for your views.

Suppose we created a `Map` class named `Map::Definitions::CommentCount`. Like with condition modules, `Lounge` will ignore the namespace of a class when generating a map method. Therefore, the following code:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  map Map::Definitions::CommentCount do
    conditions do
      written_anonymously :author => nil
      written_for_print   :issue  => not(nil)
    end
  end
end
```

This would generate `map_comment_count` and a `count_comment_count` methods on our `Article` model. We could give our view a different name by wrapping this all in the `view` method:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  view :by_popularity do
    map Map::Definitions::CommentCount
    conditions do
      written_anonymously :author => nil
      wrriten_for_print   :issue  => not(nil)
    end
  end
end
```

This will generate `map_by_popularity` and `count_by_popularity` methods on the `Article` class. Also notice that we specified `conditions` directly under the `view`, instead of wrapping it in a block under our `map`. 

## Loading data into your models

By default, `Lounge` query proxies will set the `include_docs` query string parameter to true, and it will load the value of the "doc" property in each row of results into it's corresponding model:

For example, given the following `Article` document model definition:

```ruby
class Article
  include Lounge::Document
  map :author
end
```

And the following articles:

```ruby
Article.create :id => "article1", :author => "moonmaster9000", :title => "article 1", :content => "Some long article body"
Article.create :id => "article2", :author => "moonmaster10000", :title => "article 2", :content => "Another long article body..."
```

Then calling `Article.map_by_author!` will send a request to `http://localhost:5984/cms/_design/Article/_view/by_author?reduce=false&include_docs=true`. CouchDB will return the following JSON:

```javascript
{
  "total_rows": 2,
  "offset":0,
  "rows":[
    {
      "id":"article1",
      "key":"moonmaster9000",
      "value": null,
      "doc": {
        "_id" => "article1",
        "_rev" => "1-jkfldaju328949032849032",
        "lounge_type" => "Article"
        "author" => "moonmaster9000",
        "title" => "article 1",
        "content" => "Some long article body"
      }
    },
    {
      "id":"article2",
      "key":"moonmaster10000",
      "value": null,
      "doc": {
        "_id" => "article2",
        "_rev" => "1-7373jkjkslfds28949032849032",
        "lounge_type" => "Article",
        "author" => "moonmaster10000",
        "title" => "article 2",
        "content" => "Another long article body..."
      }
    }
  ]
}
```

`Lounge` will then populate two `Article` instances with the data in the "doc" properties in this response, and return those to you. 

Sometimes, however, it's advantageous to create a map that emits some subset of values in your document; if you set `include_docs` to false, then `Lounge` will use the values in your index when loading models:

```ruby
module Map
  class Author
    include Lounge::Map

    def map
      <<-JS
      function(doc){
        if (#{conditions} and typeof doc.author !== undefined){
          emit(
            doc.author, 
            {
              "title": doc.title,
              "author": doc.author
            }
          )
        }
      }
      JS
    end
  end
end
```

Next, let's rewrite our Article definition to use this map:

```ruby
class Article
  include Lounge::Document
  include Lounge::Design

  map Map::Author
end
```

Now, `http://localhost:5984/cms/_design/Article/_view/by_label?reduce=false&include_docs=true` would return:

```javascript
{
  "total_rows": 2,
  "offset":0,
  "rows":[
    {
      "id":"article1",
      "key":"moonmaster9000",
      "value": {
        "author" => "moonmaster9000",
        "title" => "article 1"
      }
    },
    {
      "id":"article2",
      "key":"moonmaster10000",
      "value": {
        "title" => "article 2",
        "author" => "moonmaster10000"
      }
    }
  ]
}
```

Thus, if we `map_by_label` and set `include_docs` to false, `Lounge` will use the values emitted in our index to populate an anonymous document model:

```ruby
articles = Article.map_by_label.include_docs(true).execute!
articles.first.title #==> "article 1"
articles.first.author #==> "moonmaster9000"
articles.first.content #==> nil
articles.first.class #==> Lounge::AnonymousDocument
```

If we include the "loung_type" property in our map values, then `Lounge` will load those values as `Article` instances:

```ruby
module Map
  class Author
    include Lounge::Map

    def map
      <<-JS
      function(doc){
        if (#{conditions} and typeof doc.author !== undefined){
          emit(
            doc.author, 
            {
              "lounge_type": doc.lounge_type,
              "title": doc.title,
              "author": doc.author
            }
          )
        }
      }
      JS
    end
  end
end
```

Now:

```ruby
Article.map_by_label.include_docs(false).first.class #==> Article
```

