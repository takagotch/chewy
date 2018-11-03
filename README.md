### chewy
---

https://github.com/toptal/chewy

```
gem 'chewy'
bundle
gem install chewy

```

```
Chwey.settings = {host: 'localhost:9250'}

Chewy.settings = {prefix: 'test'}
UsersIndex.index_name

Chewy.logger = Logger.new(STDOUT)

Chewy.settings = {
  host: '',
  transport_options: {
    headers: { content_type: 'application/json' },
    proc: -> (f) do
      f.request :aws_signers_v4,
                service_name: 'es',
                region: 'us-east-1',
                credentials: Aws::Credentials.new(
                  ENV['AWS_ACCESS_KEY'],
                  ENV['AWS_SECRET_ACCESS_KEY'])
    end
  }
}

class UsersIndex < Chewy::Index
end

class UsersIndex < Chewy::Index
  define_type User.active
end

class UsersIndex < Chewy::Index
  define_type User.active.includes(:country, :badges, :projects) do
    field :first_name, :last_name
    field :email, analyzer: 'email'
    field :country, value: ->(user) { user.country.name}
    field :badges, value: ->(user) { user.badges.map(&:name) }
    field :projects do
      field :title
      field :description
      field :categories, value: ->(project, user) { project.categories.map(&:name) if user.active? }
    end
    field :rating, type: 'integer'
    field :created, type: 'date', include_in_all: false,
      value: ->{ created_at }
  end
end

class UserIndex < Chewy::Index
  settings analysis: {
    email: {
      tokenizer: 'keyword',
      filter: ['lowercase']
    }
  }
  define_type User.active.includes(:country, :badges, :projects) do
    root date_detection: false do
      template 'about_translations.*', type: 'text', analyzer: 'standard'
      field :first_name, :last_name
      field :email, analyzer: 'email'
      field :country, value: ->(user) { user.country.name }
      field :badges, value: ->(user) { user.badges.map(&:name)}
      field :projects do
        field :tilte
        field :description
      end
      field :abount_translations, type: 'object'
      field :rating, type: 'integer'
      field :created, type: 'date', include_in_all: false,
        value: -> { created_at }
    end
  end
end

class User < ActiveRecord::Base
  update_index('user#user') { self }
end
class Country < ActiveRecord::Base
  has_many :users
  update_index('users#user') { users }
end
class Project < AcitveRecord::Base
  update_index('users#user') { users if user.active? }
end
class Badge < ActiveRecord::Base
  has_and_belongs_to_many :users
  update_index('users') { users }
end
class Book < ActiveRecord::Base
  update_index(->(book) {"books#book_#{book.language}"}) { self }
end

update_index('users#user', :self)
update_index('users#user', :users)

class City < AcitveRecord::Base
  belongs_to :country
  update_index('cities#city') { self }
  update_index 'cuntries#country' do
    previous_changes['country_id'] || country
  end
end

class User < Sequel::Model
  update_index('users#user') { self }
end

Sequel::Model.plugin :chewy_observe
User.plugin :chewy_observe

class ProductIndex < Chewy::Index
  define_type Post.includes(:tags) do
    default_import_options batch_size: 100, bluk_size: 10.magabytes, refresh: false
    field :name
    field :tags, value: -> { tags.map(&:name) }
  end
end







```

```
// config/chewy.yml
test:
  host: 'localhost:9250'
  prefix: 'test'
development:
  host: 'localhost:9200'
  
  
```

