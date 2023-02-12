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