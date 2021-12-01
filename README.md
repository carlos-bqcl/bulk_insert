# BulkInsert

A little ActiveRecord extension for helping to insert lots of rows in a
single insert statement.

## Installation

Add it to your Gemfile:

```ruby
gem 'bulk_insert'
```

## Usage

BulkInsert adds a new class method to your ActiveRecord models:

```ruby
class Book < ActiveRecord::Base
end

book_attrs = ... # some array of hashes, for instance
Book.bulk_insert do |worker|
  book_attrs.each do |attrs|
    worker.add(attrs)
  end
end
```

All of those `#add` calls will be accumulated into a single SQL insert
statement, vastly improving the performance of multiple sequential
inserts (think data imports and the like).

If you don't like using a block API, you can also simply pass an array
of rows to be inserted:

```ruby
book_attrs = ... # some array of hashes, for instance
Book.bulk_insert values: book_attrs
```

By default, the columns to be inserted will be all columns in the table,
minus the `id` column, but if you want, you can explicitly enumerate
the columns:

```ruby
Book.bulk_insert(:title, :author) do |worker|
  # specify a row as an array of values...
  worker.add ["Eye of the World", "Robert Jordan"]

  # or as a hash
  worker.add title: "Lord of Light", author: "Roger Zelazny"
end
```

It will automatically set `created_at`/`updated_at` columns to the current
date, as well.

```ruby
Book.bulk_insert(:title, :author, :created_at, :updated_at) do |worker|
  # specify created_at/updated_at explicitly...
  worker.add ["The Chosen", "Chaim Potok", Time.now, Time.now]

  # or let BulkInsert set them by default...
  worker.add ["Hello Ruby", "Linda Liukas"]
end
```

Similarly, if a value is omitted, BulkInsert will use whatever default
value is defined for that column in the database:

```ruby
# create_table :books do |t|
#   ...
#   t.string "medium", default: "paper"
#   ...
# end

Book.bulk_insert(:title, :author, :medium) do |worker|
  worker.add title: "Ender's Game", author: "Orson Scott Card"
end

Book.first.medium #-> "paper"
```

By default, the batch is always saved when the block finishes, but you
can explicitly save inside the block whenever you want, by calling
`#save!` on the worker:

```ruby
Book.bulk_insert do |worker|
  worker.add(...)
  worker.add(...)

  worker.save!

  worker.add(...)
  #...
end
```

That will save the batch as it has been defined to that point, and then
empty the batch so that you can add more rows to it if you want. Note
that all records saved together will have the same created_at/updated_at
timestamp (unless one was explicitly set).

### Batch Set Size

By default, the size of the insert is limited to 500 rows at a time.
This is called the _set size_. If you add another row that causes the
set to exceed the set size, the insert statement is automatically built
and executed, and the batch is reset.

If you want a larger (or smaller) set size, you can specify it in
two ways:

```ruby
# specify set_size when initializing the bulk insert...
Book.bulk_insert(set_size: 100) do |worker|
  # ...
end

# specify it on the worker directly...
Book.bulk_insert do |worker|
  worker.set_size = 100
  # ...
end
```

### Insert Ignore

By default, when an insert fails the whole batch of inserts fail. The
_ignore_ option ignores the inserts that would have failed (because of
duplicate keys or a null in column with a not null constraint) and
inserts the rest of the batch.

This is not the default because no errors are raised for the bad
inserts in the batch.

```ruby
destination_columns = [:title, :author]

# Ignore bad inserts in the batch
Book.bulk_insert(*destination_columns, ignore: true) do |worker|
  worker.add(...)
  worker.add(...)
  # ...
end
```

### Update Duplicates (MySQL, PostgreSQL)

If you don't want to ignore duplicate rows but instead want to update them
then you can use the _update_duplicates_ option. Set this option to true
(MySQL) or list unique column names (PostgreSQL) and when a duplicate row
is found the row will be updated with your new values.
Default value for this option is false.

You can optionally declare specific column list for update duplicates
statement. Use the _update_columns_ option, then only these columns will be
updated. Be default, if option _update_columns_ not passed, used column list
from _column_names_ option.

```ruby
destination_columns = [:title, :author]
update_columns = [:title]

# Update duplicate rows (MySQL)
Book.bulk_insert(*destination_columns, update_duplicates: true, update_columns: update_columns) do |worker|
  worker.add(...)
  worker.add(...)
  # ...
end

# Update duplicate rows (PostgreSQL)
Book.bulk_insert(*destination_columns, update_duplicates: %w[title], update_columns: update_columns) do |worker|
  worker.add(...)
  # ...
end
```

### Return Primary Keys (PostgreSQL, PostGIS)

If you want the worker to store primary keys of inserted records, then you can
use the _return_primary_keys_ option. The worker will store a `result_sets`
array of `ActiveRecord::Result` objects. Each `ActiveRecord::Result` object
will contain the primary keys of a batch of inserted records.

```ruby
worker = Book.bulk_insert(*destination_columns, return_primary_keys: true) do
|worker|
  worker.add(...)
  worker.add(...)
  # ...
end

worker.result_sets
```

## Ruby and Rails Versions Supported

> :warning: The scope of this gem may be somehow covered natively by the `.insert_all` API
> introduced by [Rails 6](https://apidock.com/rails/v6.0.0/ActiveRecord/Persistence/ClassMethods/insert_all).
> This gem represents the state of art for rails version < 6 and it is still open to
> further developments for more recent versions.

The current CI prevents regressions on the following versions:

ruby / rails | `~>3` | `~>4` | `~>5` | `~>6`
:-----------:|-------|-------|-------|------
2.2          |  yes  |  yes  |  no   |  no
2.3          |  yes  |  yes  |  yes  |  no
2.4          |  no   |  yes  |  yes  |  no
2.5          |  no   |  no   |  yes  |  yes
2.6          |  no   |  no   |  yes  |  yes
2.7          |  no   |  no   |  yes  |  yes

The adapters covered in the CI are:
* sqlite
* mysql
* postgresql


## License

BulkInsert is released under the MIT license (see MIT-LICENSE) by
Jamis Buck (jamis@jamisbuck.org).
