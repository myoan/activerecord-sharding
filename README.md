# ActiveRecord::Sharding

[![Build Status](https://travis-ci.org/hirocaster/activerecord-sharding.svg)](https://travis-ci.org/hirocaster/activerecord-sharding) [![Coverage Status](https://coveralls.io/repos/hirocaster/activerecord-sharding/badge.svg?branch=master&service=github)](https://coveralls.io/github/hirocaster/activerecord-sharding?branch=master) [![Code Climate](https://codeclimate.com/github/hirocaster/activerecord-sharding/badges/gpa.svg)](https://codeclimate.com/github/hirocaster/activerecord-sharding) [![Dependency Status](https://gemnasium.com/hirocaster/activerecord-sharding.svg)](https://gemnasium.com/hirocaster/activerecord-sharding)

Simple Database Sharding in ActiveRecord.

ActiveRecord object distributed multiple databases by modulo.
Modulo target is id, generated by sequencer database.

## Support database

- MySQL

## Installation

Add this line to your application's Gemfile:

    gem 'activerecord-sharding'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install activerecord-sharding

## Usage

Add database connections to your application's config/database.yml:

```yaml
default: &default
  adapter: mysql2
  encoding: utf8
  pool: 5
  username: root
  password:
  host: localhost

user_sequencer:
  <<: *default
  database: user_001
  host: localhost

user_001:
  <<: *default
  database: user_001
  host: localhost

user_002:
  <<: *default
  database: user_002
  host: localhost

user_003:
  <<: *default
  database: user_003
  host: localhost
```

Add this example  your application's config/initializers/active_record_sharding.rb:

```ruby
ActiveRecord::Sharding.configure do |config|
  config.define_sequencer(:user) do |sequencer|
    sequencer.register_connection(:user_sequencer)
    sequencer.register_table_name('user_id')
  end

  config.define_cluster(:user) do |cluster|
    cluster.register_connection(:user_001)
    cluster.register_connection(:user_002)
    cluster.register_connection(:user_003)
  end
end
```

- define user cluster config
- define user sequencer config

### Model

app/model/user.rb

```ruby
class User < ActiveRecord::Base
  include ActiveRecord::Sharding::Model
  use_sharding :user, :modulo # shard name, algorithm
  define_sharding_key :id

  include ActiveRecord::Sharding::Sequencer
  use_sequencer :user

  before_put do |attributes|
    attributes[:id] = next_sequence_id unless attributes[:id]
  end
end
```


### Create sequencer databases

```ruby
$ rake active_record:sharding:sequencer:setup
```

### Create cluster dtabases

```ruby
$ rake active_record:sharding:setup
```

and, migrations all cluster databases.

### in applications

#### Create

using `#put!` method.

```ruby
user = User.put! name: 'foobar'
```

if transaction to shard database and nest other database transactions.

```ruby
User.put!(name: 'foobar') do |new_user| # transaction for user shard database
  OTHER_MODEL.transaction do
    OTHER_MODEL.create!(user_id: new_user.id)
  end
end
```

returns User new object.

#### Select Query

```ruby
sharding_key = user.id
User.shard_for(sharding_key).where(name: 'foorbar')
```

`sharding_key` is your define_syarding_key.(example is User Object id)

`#sahrd_for` is returns User class.

for all shards query

```ruby
User.all_shards.flat_map { |model| model.find_by(name: 'foorbar') }.compact
```

#### Association/Relation

if use database association/relation in sharding databases.

Please, don't use ActiveRecord standard associations/relation features(has_may, has_one, belongs_to... etc).because, it using `ActiveRecord::Base.connection`(not sharding databases conneciton).

Please, manually add association/relation methods.

Bad sample

```
class User < ActiveRecord::Base
  has_many :items # connect to not sharding databases(default database)

  include ActiveRecord::Sharding::Model
  use_sharding :user, :modulo
  define_sharding_key :id
  # (snip)
end
```

Manually add method

```
class User < ActiveRecord::Base
  def items
    return [] unless id
    Item.shard_for(id).where(user_id: id).all
  end

  include ActiveRecord::Sharding::Model
  use_sharding :user, :modulo
  define_sharding_key :id
  # (snip)
end
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/hirocaster/activerecord-sharding.
