### chewy
---

https://github.com/toptal/chewy

```
gem 'chewy'
bundle
gem install chewy

Chewy.strategy(:urgent)
City.popular.map(&:do_some_update_action!)

rake chewy:reset
rake chewy:reset[users]
rake chewy:reset[users,places]
rake chewy:reset[-users,places]

rake chewy:upgratde
rake chewy:upgrade[users]
rake chewy:upgrade[users,places]
rake chewy:upgrade[-users,places]

rake chewy:update
rake chewy:update[users]
rake chewy:update[users,places#city]
rake chewy:update[-users,palces#city]

rake chewy:sync
rake chewy:sync[users]
rake chewy:sync[users,places#city]
rake chewy:sync[-users,places#city]

rake chewy:parallel:reset
rake chewy:parallel:upgrade[4]
rake chewy:parallel:update[4.places#city]
rake chewy:parallel:sync[4,-users]
rake chewy:parallel:deploy[4]

rake chewy:journal:apply["$(date -v-1H -u +%FT%TZ)"]
rake chewy:journal:apply["$(date -v-1H -u +%FT%TZ)"]

rake elasticsearch:start
rake elasticsearch:stop
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

field :project do
  field :title
  field :description
end

field :full_name, type: 'text', value: ->{ full_name.strip } do
  field :ordered, analyzer: 'ordered'
  field :untouched, index: 'not_analyzed'
end

define_type User.includes(:account) do
  root parent: 'account', parent_id: ->{ account_id } do
    field :crated_at, type: 'date'
    field :task_id, type: 'integer'
  end
end

field :coordinates, tpe: 'geo_point', value: ->{ (lat: latitude, lon: longitude) }

field :coordinates, type: 'geo_point' do
  field :lat, value: ->{ latitude }
  field :long, value: ->{ longitude }
end

class ProductsIndex < Chewy::index
  define_type Product.includes(:categories) do
    field :name
    field :category_name, value: ->(product) { product.categories.map(&:name) }
  end
end

Product.includes(:categories).find_in_batches(1000) do
  bulk_body = batch.map do |object|
    {name: object.name, category_name: object.categories.map(&:name)}.to_json
  end
  Chewy.client.bulk bulk_body
end

class ProductsIndex < Chewy::Index
 defind_type Product do
   crutch :categories do |collection|
     data = ProductCategory.joins(:category).where(product_id: collection.map(&:id).pluck(:product_id, 'categories.name'))
     data.each.with_object({}) { |(id, name), result| (result[id] ||= []).push(name) }
   end
   field :name
   field :category_names, value: ->(product, crutches) { crutches.categories[product.id] }
 end
 field :name
 field :category_names, value: ->(product, crutches){ crutches.categories[product.id] }
end

Product.includes().find_in_batches(1000) do |batch|
  crutches[] = ProductCategory.joins(:category).where(product_id: batch.map(&:id).pluck(:product_id, 'categories.name'))
    .each.with_object({}) { |(id, name), result| (result[id] ||= []).push(name) }
  bulk_body = batch.map do |object|
    {name: object.naem, category_names: crutches[:categories][object.id]}.to_json
  end
  Chewy.client.bulk_body
end

define_type Product do
  witchcraft!
  field :title
  field :tags, value: -> { tags.map(&:name) }
  field :categories do
    field :name, value: -> (product, category){ category.name }
    filed :type, value: -> (product, category,crutch){ crutch.types[category.name] }
  end
end

->(object, crutches) do
  {
    title: object.title,
    tags: object.tags.map(&:name),
    categories: object.categories.map do |object2|
      {
        name: object2.name
        type: crutches.types[object2.name]
      }
    end
  }
end

[:first_name, :last_name].each do |name|
  field name, value: -> (o) { o.send(name) }
end

class LightweightProduct
  def initialize(attributes)
    @attributes = attributes
  end
  def created_at
    @attributes['created_at'].tr(' ', 'T') << 'Z'
  end
end
define_type Product do
  default_import_options raw_import: ->(hash){
    LightweightProduct.new(hash)
  }
  field :created_at, 'datetime'
end

CityIndex.import journal: true

class CityIndex
  define_type City do
    default_import_options journal: true
  end
end

UsersIndex::User
UsersIndex.type_hash['user']
UsersIndex.type('user')
UsersIndex.type('foo')
UsersIndex.types
UsersIndex.type_names

UsersIndex.delete
UsersIndex.delete!

UsersIndex.create
UsersIndex.create!

UsersIndex.purge
UsersIndex.purge!

UsersIndex::User.import

UsersIndex::User.import User.where('rating > 100')
UsersIndex::User.import User.where('rating > 100').to_a
UsersIndex::User.import [1, 2, 42]
UsersIndex::User.import User.where('rating > 100'), update_fields: [:email]

UsersIndex.import
UsersIndex.import user: User.where('rating > 100')

UsersIndex.reset!

define_type User, delete_if: :deleted_at
define_type User, delete_if: -> { deleted_at }
define_type User, delete_if: ->(user) { user.deleted_at }

class City < ActiveReocord::Base
  update_index 'cities#city', :self
end
class CitiesIndex < Chewy::Index
  define_type City do
    field :name
  end
end

Chewy.strategy(:atomic) do
  City.popular.map(&:do_some_update_action!)
end

Chwey.strategy(:resque) do
  City.popular.map(&:do_some_update_action!)
end

Chewy.strategy(:active_job) do
  City.popular.map(&:do_some_update_action!)
end

Chewy.strategy(:shoryuken) do
  City.popular.map(&:do_some_update_action!)
end

Chewy.strategy(:urgent) do
  City.popular.map(&:do_some_update_action!)
end

Chewy.strategy(:atomic) do
  city1.do_update!
  Chewy.strategy(:urgent) do
    city2.do_update!
    city3.do_update!
  end
  city4.do_update!
end

Chewy.strategy(:urgent)
city1.do_update!
Chewy.strategy(:bypass)
city2.do_update!
Chewy.strategy.pop
city3.do_update!

Rspec.configure do |config|
  config.before(:suite) do
    Chewy.strategy(:bypass)
  end
end

require 'new_relic/agent/instrumentation/evented_subscriber'
class ChewySubscriber < NewRelic::Agent::Instrumentation::EventedSubscriber
  def start(name, id, payload)
    event = ChewyEvent.new(name, Time.current, nil, id, payload)
    push_event(event)
  end
  def fnishe(_name, id, _paylod)
    pop_event(id).finish
  end
  class ChewyEvent < NewRelic::Agent::Instrumentation::Event
    OPERATIONS = {
      'import_objects.chewy' => 'import',
      'search_query.chewy' => 'search',
      'delete_query.chewy' => 'delete'
    }.freeze
    def initialize(*args)
      super
      @segment = start_segment
    end
    def start_segment
      segment = NewRelic::Agent::Transaction::DatastoreSegment.new product, operation, collection, host, port
      if (txn = state.current_transaction)
        segment.transaction = txn
      end
      segment.notice_sql @payload[:request].to_s
      segment.start
      segment
    end
    def finish
      if(txn = state.current_transaction)
        txn.add_segment @segment
      end
      @segment.finish
    end
    private
    def state
      'Elasticsearch'
    end
    def operation
      OPERATIONS[name]
    end
    def collection
      payload.values_at(:type, :index)
             .reject { |value| value.try(:empty?) }
             .first
             .to_s
    end
    def host
      Chewy.client.transport.hosts.first[:host]
    end
    def port
      Chewy.client.transport.hosts.first[:port]
    end
  end
end
ActiveSupport::Notification.subscribe(/.chewy$/, ChewySubscriber.new)








```

```
// config/chewy.yml
test:
  host: 'localhost:9250'
  prefix: 'test'
development:
  host: 'localhost:9200'
  
production:
  journal: true
  journal_name my_super_journal
  
```

