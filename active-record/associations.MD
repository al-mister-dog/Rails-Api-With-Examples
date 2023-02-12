# Associations
Associations are a set of macro-like[^1] class methods for tying objects together through foreign keys. They express relationships like “Project has one Project Manager” or “Project belongs to a Portfolio”. Each macro adds a number of methods to the class which are specialized according to the collection[^2] or association symbol[^3] and the options hash[^4] It works much the same way as Ruby's own attr* methods.[^5]

```ruby
class Project < ActiveRecord::Base
  belongs_to              :portfolio
  has_one                 :project_manager
  has_many                :milestones
  has_and_belongs_to_many :categories
end
```
The project class now has the following methods (and more) to ease the traversal and manipulation of its relationships:

* Project#portfolio[^6], Project#portfolio=(portfolio), Project#reload_portfolio

* Project#project_manager, Project#project_manager=(project_manager), Project#reload_project_manager

* Project#milestones.empty?, Project#milestones.size, Project#milestones, Project#milestones<<(milestone), Project#milestones.delete(milestone), Project#milestones.destroy(milestone), Project#milestones.find(milestone_id), Project#milestones.build, Project#milestones.create

* Project#categories.empty?, Project#categories.size, Project#categories, Project#categories<<(category1), Project#categories.delete(category1), Project#categories.destroy(category1)





### Unsaved objects and associations
You can manipulate objects and associations before they are saved to the database, but there is some special behavior you should be aware of, mostly involving the saving of associated objects.

You can set the :autosave option on a has_one, belongs_to, has_many, or has_and_belongs_to_many association. Setting it to true will always save the members, whereas setting it to false will never save the members. More details about :autosave option is available at AutosaveAssociation.

#### One-to-one associations
- Assigning an object to a has_one association automatically saves that object and the object being replaced (if there is one), in order to update their foreign keys - except if the parent object is unsaved (new_record? == true).

```ruby
class User < ApplicationRecord
  has_one :address
end

user = User.first
address = Address.new(street: "123 Main St", city: "Anytown", state: "CA")

user.address = address
user.save # This will save the user and the address, updating the foreign key on the address record to match the ID of the user.

# Alternatively, you can build a new address without saving it:
user.build_address(street: "456 Main St", city: "Anytown", state: "CA")
```

- If either of these saves fail (due to one of the objects being invalid), an ActiveRecord::RecordNotSaved exception is raised and the assignment is cancelled.
```ruby
class User < ApplicationRecord
  has_one :address
end

user = User.first
address = Address.new(street: "", city: "Anytown", state: "CA")

# This will raise an ActiveRecord::RecordNotSaved exception because the address is invalid (street is blank).
user.address = address
```

- If you wish to assign an object to a has_one association without saving it, use the #build_association method (documented below). The object being replaced will still be saved to update its foreign key.
```ruby
class User < ApplicationRecord
  has_one :address
end

user = User.first

# This will build a new address object and associate it with the user, but it will not save it to the database.
new_address = user.build_address(street: "456 Main St", city: "Anytown", state: "CA")
```

- Assigning an object to a belongs_to association does not save the object, since the foreign key field belongs on the parent. It does not save the parent either.
```ruby
class Address < ApplicationRecord
  belongs_to :user
end

address = Address.first
user = User.new(name: "John Doe")

address.user = user
address.save # This will save the address, updating the foreign key on the address record to match the ID of the user, but it will not save the user.
```

#### Collections
- Adding an object to a collection (has_many or has_and_belongs_to_many) automatically saves that object, except if the parent object (the owner of the collection) is not yet stored in the database.
```ruby
class User < ApplicationRecord
  has_many :posts
end

user = User.first
post = Post.new(title: "My first post", body: "Lorem ipsum...")

# This will add the post to the user's collection of posts and save it to the database.
user.posts << post

# Alternatively, you can use the push method:
user.posts.push(post)
```
If saving any of the objects being added to a collection (via push or similar) fails, then push returns false.
```ruby
class User < ApplicationRecord
  has_many :posts
end

user = User.first
post = Post.new(title: "", body: "Lorem ipsum...")

# This will return false because the post is invalid (title is blank).
result = user.posts << post
```

If saving fails while replacing the collection (via association=), an ActiveRecord::RecordNotSaved exception is raised and the assignment is cancelled.
```ruby
class User < ApplicationRecord
  has_many :posts
end

user = User.first
post1 = Post.new(title: "My first post", body: "Lorem ipsum...")
post2 = Post.new(title: "", body: "Lorem ipsum...")

# This will raise an ActiveRecord::RecordNotSaved exception because one of the posts is invalid (title is blank).
user.posts = [post1, post2]
```

You can add an object to a collection without automatically saving it by using the collection.build method (documented below).
```ruby
class User < ApplicationRecord
  has_many :posts
end

user = User.first

# This will build a new post object and add it to the user's collection of posts, but it will not save it to the database.
new_post = user.posts.build(title: "My first post", body: "Lorem ipsum...")
```

All unsaved (new_record? == true) members of the collection are automatically saved when the parent is saved.
```ruby
class User < ApplicationRecord
  has_many :posts
end

user = User.first
post = Post.new(title: "My first post", body: "Lorem ipsum...")

user.posts << post

# This will save the user and all unsaved members of its collection of posts.
user.save
```

[^1]: In Ruby on Rails, a macro is a method that generates other methods dynamically. Macros can be used to generate methods that provide common functionality across multiple models, making it easier to manage and maintain your code.

[^2]: In Ruby on Rails, a collection refers to a group of related objects. For example, a User model might have many Posts, and the posts belonging to a user can be thought of as a collection of posts. When you use the has_many macro in Active Record, you are creating a one-to-many relationship between two models, with one model (the user) having a collection of related objects (the posts).

[^3]: An association symbol is a symbol that represents the type of association being created between two models. For example, the :posts symbol in has_many :posts represents the collection of posts that belong to a user. Other common association symbols include :belongs_to, :has_one, and :has_many :through.

[^4]: An options hash is a hash that is passed as an argument to an Active Record macro and provides additional information about the association being created. For example, you might specify the :foreign_key option to specify a custom foreign key to use in the association. Options hashes are used to fine-tune the behavior of associations in Active Record.

[^5]: Ruby's own attr* methods are methods that are automatically generated by Ruby to define accessors (getters and setters) for instance variables. For example, the attr_reader method generates a getter method that allows you to retrieve the value of an instance variable, while attr_writer generates a setter method that allows you to set the value of an instance variable. Ruby's own attr* methods are similar to the methods generated by Active Record macros in that they dynamically generate methods for you, but they are used for a different purpose (defining accessors for instance variables) and in a different context (Ruby classes in general, rather than just Active Record models).

[^6]: In the context of Ruby on Rails documentation, the "#" symbol is used to separate the class name from the instance method name. In this case, "Project#milestones.create" means that the create method is an instance method of the milestones method, which is a part of the Project class.

An instance method is a method that can be called on an instance of a class, as opposed to a class method, which can be called on the class itself. In other words, an instance method operates on an instance of the class and can access and modify the state of that instance, while a class method operates on the class and doesn't have access to any specific instance.