



# Starting out with LocomotiveCMS.

## Gems
LocomotiveCMS uses **Carrierwave** gem for attachment management. 
LocomotiveCMS uses MongoDB as its database, and its datamapper (Object-Document Mapping) is **Mongoid** as an alternative for usual ActiveRecord library that is used as an ORM (Object-Relational Mapping) for relational databases correspondingly (MySQL, PostgreSQL).
Still, Mongoid uses some parts of ActiveModel - eg Mongoid includes ActiveModel::Validations to provide us with usual `validates_presence_of :name` syntax.

## MongoDB essentials
MongoDB is a NoSQL database. After you install it, you should start a server with `mongod`. Then you will be able to access a console with `mongo`.
There are many types of NoSQL databases: key-value (Redis, CouchDB), column (Cassandra), document-oriented (MongoDB), and a few more. 
MongoDB is a document-oriented db and it uses a different from SQL databases syntax for similar concepts. For example, what we call a document in MongoDB would be called a row in SQL database. You can learn about other terminology parallels here: http://docs.mongodb.org/manual/reference/sql-comparison/.

Why use new terminology (collection vs. table, document vs. row and field vs. column)? Is it just to make things more complicated? The truth is that while these concepts are similar to their relational database counterparts, they are not identical. The core difference comes from the fact that relational databases define columns at the table level whereas a document-oriented database defines its fields at the document level. That is to say that each document within a collection can have its own unique set of fields.

Some other interesting things about MongoDB:

- it doesn't support table joins. dont be very afraid to store repeated data.
- given that collections don’t enforce any schema, it's entirely possible to build a system using a single collection with a mishmash of documents (but it would be a very bad idea)
- it doesn't support transactions

more on mongodb: https://www.mongodb.com/blog/post/thinking-documents-part-1?jmp=docs&_ga=1.54024672.951253070.1435561573

links used: http://openmymind.net/mongodb.pdf

## MongoDB console
To get an idea on how it works in practice, input `use hi` in your mongo console. It doesn't matter that the database doesn't really exist yet. The first collection that we create will also create the actual 'hi' database.
To list collections (==tables) in our 'hi' database, input `db.getCollectionNames()`. Since we have no tables, this should return `[]`. To create a collection, we can just create a new document, and this will automatically create its table. So, to create unicorns database with a new record, use:

    db.unicorns.insert({name: 'Aurora', age: 200}) #will create a collection unicorns 
    #unless it's already created and create a document with fields 'name' and 'age'

Argument passed to the insert command is in a JSON format. Internally MongoDB uses BSON (Binary JSON) to store data.

    db.unicorns.find() #will return all documents from unicorns collection

will return all documents (==rows) unicorns collection (==table) currently has.

You may notice that documnent returned after executing `db.unicorns.find()` has an **_id** field in it: `{ "_id" : ObjectId("559881b62b9a63424d1c43d6"), "name" : "Aurora", "age" : 200 }`. It was created automatically to provide a document with unique identifier.

If we execute `db.unicorns.insert({home: 'forest'})` now, we will see by `db.unicorns.find()` that this document was gladly added even though its fields (==columns) are completely different from the previous record.

    db.unicorns.remove({}) #will delete all documents from unicorns collection
    db.unicorns.remove({name: 'evelyn'}) #will delete records with name: 'evelyn'


## Mongoid

Follow this tutorial to get a feeling on how to work with Mongoid gem: http://railscasts.com/episodes/238-mongoid. 
Mongoid supports both relational-style associations (through post_id, eg) and embedded associations. However, [it's not recommended to use relational associations extensively](http://mongoid.org/en/mongoid/docs/tips.html#relational_associations). 
What embedded association basically is, is a document embedded in another document. You could think it's a bad style from relation database point of view, but it's actually a preferred way of organizing a document-oriented db. The reasoning behind it is as follows: 
 - MongoDB provides no transactions, and no join support, so it is impossible to ensure that the database is kept in a consistent state with regards to referencing documents from one collection to another
 - without support for joins, you will find that the number of database queries executed grows in order to retrieve documents with their associations.

Data modeled with these classes:

	class Article
          field :title #defaults to ", type: String"
          field :published_on, type: Date
	  embeds_many :comments
	end

	class Comments
          field :content
	  embedded_in :article, :inverse_of => :comments
	end

may look like this upon retrieval (of Article instance):

    {
	title: 'Apples',
	published_on: Fri, 10 Jul 2015
	comments: [
	            { content: 'are good' },
	            { content: 'are bad' }
	          ]
    }

Valid types for fields somewhat correspond to Ruby's classes: ["Array", "BigDecimal", "Boolean", "Date", "DateTime", "Float", "Hash", "Integer", "BSON::ObjectId", "Moped::BSON::Binary", "Range", "Regexp", "String", "Symbol", "Time", "TimeWithZone"]. So that if you'd like your field to be of type Text, use String type as it is of variable length, just like in Ruby.

As already mentioned, MongoDB supports **dynamic fields** - if in your console you'll write

    db.unicorns.insert({home: 'forest'})

it will automatically create a `home` field for `unicorn` collection. Mongoid supports dynamic attributes out of the box as well, so that if we try:
    
    Article.find_by(title: 'Apples')[:gender] = 'Female'

, an article with gender 'Female' will be initialized.
To save it, use usual `article.save`. This will, however, not work with `.gender` accessor.

A few comparisons of ActiveRecord vs Mongoid methods:

| ActiveRecord                                                                                         | Mongoid                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Article.all` => ActiveRecord::Relation object, that will list all rows                              | `Article.all` => Mongoid::Criteria object. It's lazily evaluated (only touches db when needed), so to get all documents, use, eg, Article.all.to_a  |
| `Article.all.each {|article| ...}`                                                                  | `Article.each {|article| ...} `                                                                                                                     |
| `Article.find(12)`, will raise an error if record was not found                                                                                   | `Article.find(12)`, same                                 |
| `Article.find_by(title: 'Apples')`                                                                   | `Article.find_by(title: 'Apples')` similar as ActiveRecord's find_by (finds by title), but will raise an error if record was not found              |
| `Article.where(title: 'Apples')`                                                                     | `Article.where(title: 'Apples')`                                                                                                                    |
| `Article.create(gender: 'female')` will init and persist a row if gender column exists in this table | `Article.create(gender: 'female')` will init and persist a document even if there is no predefined field 'gender' in this collection                |
| `article.update_attributes` alias or #update                                                         | `article.update_attributes` updates attributes with validations and callbacks.                                                                      |
| `article.destroy` deletes row, runs destroy callbacks                                                | `article.destroy`, same                                                                                                                              |


## LocomotiveCMS structure

In LocomotiveCMS, model is called a Content Type. 
LocomotiveCMS mimics a usual Rails application. I allows to create a blog through user interface alone, where following analogies could be drawn:

| Usual Rails App                                             | LocomotiveCMS interface |
| ----------------------------------------------------------- | ----------------------- |
| Model                                                       | Content Type            |
| Instance of a model, a row                                  | Content Entry           |
| Fields user creates themselves (not _id, created_at, ...)   | Custom Fields           |


In reality though, every 'model' imitation created by us is an instance of `ContentType123` (where 123 is a random number) class, and every 'entry' of that model is an instance of `ContentEntry567` class.
If we look at `ContentType123.ancestors`, we'll see that this class inherits from, expectedly, `ContentType` class, inheriting all its class and instance methods.
To see which methods are available on it, we can look at the `locomotive_cms` engine source code:
`gem which locomotive_cms` will give this engine's location in your pc, and in *app/models/locomotive/content_type.rb* you'll find a `Locomotive::ContentEntry` class.

You may notice there this line:

    module Locomotive
      class ContentType
        has_many :entries, class_name: 'Locomotive::ContentEntry', dependent: :destroy
      end
    end

which gives an idea on how we get an illusion of model - model instance relationship.


## Working with source
