# ActiveRecord Enhanced SQLite3 Adapter

Enhance ActiveRecord's 7.1 SQLite3 adapter. Adds support for:

* generated columns,
* deferred foreign keys,
* `PRAGMA` tuning,
* and extension loading

## Installation

Install the gem and add to the application's Gemfile by executing:

```shell
$ bundle add activerecord-enhancedsqlite3-adapter
```

## Usage

This gem hooks into your Rails application to enhance the `SQLite3Adapter` automatically. No setup required!

Once installed, you can take advantage of the added features.

### Configuration

One optional advanced feature is to have this gem isolate reading and writing connection pools. This is useful if you have a large amount of write operations and want to avoid blocking reads.

You can configure this gem via the Rails configuration object, under the `enhanced_sqlite3` key. Currently, only 1 configuration option is available:

* `isolate_connection_pools` - Whether or not to isolate reading from writing connection pools. See [below](#isolated-connection-pools) for more information.

### Generated columns

You can now create `virtual` columns, both stored and dynamic. The [SQLite docs](https://www.sqlite.org/gencol.html) explain the difference:

> Generated columns can be either VIRTUAL or STORED. The value of a VIRTUAL column is computed when read, whereas the value of a STORED column is computed when the row is written. STORED columns take up space in the database file, whereas VIRTUAL columns use more CPU cycles when being read.

The default is to create dynamic/virtual columns.

```ruby
create_table :virtual_columns, force: true do |t|
  t.string :name
  t.virtual :upper_name, type: :string, as: "UPPER(name)", stored: true
  t.virtual :lower_name, type: :string, as: "LOWER(name)", stored: false
  t.virtual :octet_name, type: :integer, as: "LENGTH(name)"
end
```

### Deferred foreign keys

You can now specify whether or not a foreign key should be deferrable, whether `:deferred` or `:immediate`.

`:deferred` foreign keys mean that the constraint check will be done once the transaction is committed and allows the constraint behavior to change within transaction. `:immediate` means that constraint check is immediate and allows the constraint behavior to change within transaction. The default is `:immediate`.

```ruby
add_reference :person, :alias, foreign_key: { deferrable: :deferred }
add_reference :alias, :person, foreign_key: { deferrable: :deferred }
```

### `PRAGMA` tuning

Pass any [`PRAGMA` key-value pair](https://www.sqlite.org/pragma.html) under a `pragmas` list in your `config/database.yml` file to ensure that these configuration settings are applied to all database connections.

```yaml
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  pragmas:
    # level of database durability, 2 = "FULL" (sync on every write), other values include 1 = "NORMAL" (sync every 1000 written pages) and 0 = "NONE"
    # https://www.sqlite.org/pragma.html#pragma_synchronous
    synchronous: "FULL"
```

### Extension loading

There are a number of [SQLite extensions available as Ruby gems](https://github.com/asg017/sqlite-ecosystem). In order to load the extensions, you need to install the gem (`bundle add {extension-name}`) and then load it into the database connections. In order to support the latter, this gem enhances the `config/database.yml` file to support an `extensions` array. For example, to install and load [an extension](https://github.com/asg017/sqlite-ulid) for supporting [<abbr title="Universally Unique Lexicographically Sortable Identifiers">ULIDs</abbr>](https://github.com/ulid/spec), we would do:

```shell
$ bundle add sqlite_ulid
```

then

```yaml
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  extensions:
    - sqlite_ulid
```

### Isolated connection pools

By default, Rails uses a single connection pool for both reading and writing. This can lead to contention if you have a large number of write operations. This gem allows you to isolate the connection pools for reading and writing.

To enable this feature, set the `isolate_connection_pools` configuration option to `true` in your `config/environments/*.rb` file or `config/application.rb` file:

```ruby
config.enhanced_sqlite3.isolate_connection_pools = true
```

If enabled, the gem will patch your application in 3 ways:

1. define separate `reader` and `writer` database configurations
2. activate Rails' automatic role database switching middleware, defaulting all requests to the `reader` connection pool
3. patch the ActiveRecord `#transaction` method to switch to the `writer` connection pool for write operations
4. patch the ActiveRecord `#log` method to log the database name for each database operation

This feature is experimental and may not work with all Rails configurations. Please report any issues you encounter.

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake test` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and the created tag, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/fractaledmind/activerecord-enhancedsqlite3-adapter.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
