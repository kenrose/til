# Stream Data With ActiveRecord and MySQL

The usual way to process a table with millions of records in Rails is to use `find_each` or `in_batches`.  e.g.,

```ruby
Person.find_each do |person|
  person.do_awesome_stuff
end
```

Instead of loading all `Person` records like `each` does, the above fetches 1000 records at a time and paginates through the database with `ORDER BY id` and using `LIMIT` and `OFFSET`.

There are a few issues with this though:

1. Your query will be evaluated multiple times, once for each batch.
   - If your query is expensive to compute, you will end up computing it thousands of times if you have millions of records.
   - You have no consistency guarantees.  If the data set changes between queries, you get what you get.
1. You have to `ORDER BY id`.  If you don't want that, too bad, you're out of luck.

Ideally, there would be a way to run the query once and stream records directly from the database.  Though Rails can't do it directly, it is possible with MySQL and [`Mysql2::Client`](https://github.com/brianmario/mysql2#streaming).  Here's some code:

```ruby
model_class = BigRecord
relation = model_class.where(...)
query = relation.to_sql
with_raw_connection |client|
  conn_options = {
    stream: true,
	cache_rows: false,
	as: :hash,
	cast_booleans: true,
	symbolize_keys: false
   }
   rows = client.query(query, conn_options)
   rows.each do |params|
     obj = model_class.instantiate(params)
     obj.readonly! # if you want to be really safe
   end
end

# Get a new connection from connection pool
def with_raw_connection
  conn_pool = ActiveRecord::Base.establish_connection
  begin
    conn = conn_pool.checkout
    raw_connection = conn.raw_connection
    yield raw_connection  # this is an instance of Mysql2::Client
  ensure
    conn_pool.checkin(conn)
  end
end
```

[`instantiate`](https://api.rubyonrails.org/v6.0.0/classes/ActiveRecord/Persistence/ClassMethods.html#method-i-instantiate) is wonderful and built for exactly this case.  If you happen to have additional columns in your query, they'll be added as attributes to the objects.  For example, if your query is:

```sql
SELECT big_records.*, 42 as meaning_of_life
FROM big_records
```

then every object you instantiate will have

```ruby
obj.meaning_of_life
obj.meaning_of_life=
obj[:meaning_of_life]
```

set properly.

I wrote a simple `with_raw_connection` that checks out a connection from ActiveRecord's connection pool instead of using `ActiveRecord::Base::connection` directly.  Rails needs to execute other queries to, for example, load table definitions or load associated records.  Unless you've set something differently, Rails will use `ActiveRecord::Base::connection` for these queries.  Unfortunately, in MySQL, if you try to execute a `SELECT` query on a connection while you also have a streaming query executing, you get this error:

```
ActiveRecord::StatementInvalid (Mysql2::Error: Commands out of sync; you can't run this command now: ...
```

There's also [`ConnectionPool.with_connection`](https://api.rubyonrails.org/v6.0.0/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html#method-i-with_connection), but it can sometimes return `ActiveRecord::Base.connection`, which we don't want:

> If a connection obtained through connection or with_connection methods already exists yield it to the block.


## References

- [Exporting significant SQL reports with ActiveRecord](https://getaround.tech/streaming-raw-sql-results-with-active-record/) (2018)
- [kaspernj/active-record-streamer](https://github.com/kaspernj/active-record-streamer) (2016)
   Old library with a similar rationale.  Handles object batching and preloading, but only works for Rails 4 and below.
- Rails Issue: [Implement find_each using cursors on Postgres](https://github.com/rails/rails/issues/28085) (2017)
- [`ActiveRecord::Persistence::instantiate`](https://api.rubyonrails.org/v6.0.0/classes/ActiveRecord/Persistence/ClassMethods.html#method-i-instantiate)
- Rails Issue: [A question about cursors support](https://github.com/rails/rails/issues/29648) (2017)
