---
layout: post
headline: "Eight ways to write a better data importer"
title: "Eight ways to write a better data importer"
description:
category: programming
tags: [importers, etl, ruby, rails]
modified: 2014-04-14
image:
  feature: ship-containers-2.jpg
  credit: Jim Bahn
  creditlink: https://flic.kr/p/qSbck
comments: false
share: true
---

Web applications do not live in a vacuum. Most of the web applications we write today need to operate on data from a variety of sources:

- CSV files from a customer-managed Excel Worksheets
- CSV or XML files dumped from other databases
- Web APIs
- Scraped web pages

To get this data into our system, we need to write importers, more technically known as ETL's - [extract, transform, and load](https://en.wikipedia.org/wiki/Extract,_transform,_load). These importers are often implemented as some form of command line script or rake task, and initiated either manually for one-off loads, or periodically through a crontab or other scheduler. They may be short running, completing in seconds, or long running, taking hours or days.

Over the years at [Mudbug Media](http://mudbugmedia.com), we've written _a lot_ of importers. Along the away we've encountered a gamut of common pain points and mistakes. For one of our annual department goals, we've organized them into a checklist to review when writing new importers. Most of these examples are oriented specifically for Ruby on Rails ecosystem, but can apply to most languages and frameworks.

### 1. Decouple Your Logic

Follow the [single responsibility principle](http://en.wikipedia.org/wiki/Single_responsibility_principle): Implement each importer in its own dedicated class. It's common to see importers stuffed into its corresponding model as a static method, such as:

{% highlight ruby %}
class Product < ActiveRecord::Base
  def self.import
    # A whole lot of logic here...
  end
end
{% endhighlight %}

Most [ORM](http://en.wikipedia.org/wiki/Object-relational_mapping)-based model classes (like ActiveRecord) already have too many responsibilities between representing data and how it gets stored to the database. Don't pile on the [additional burden](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/) of importing data from yet another source. Multiple responsibilities in a single class leads to code that's hard to read and hard to refactor. Implement your importers into their own [service](http://stevelorek.com/service-objects.html) classes, e.g. a `ProductImporter` class. If you have multiple importers of a similar style, have them inherit from a common parent class to provide some consistency. In Rails, don't be afraid to store them in a new more-communicative directory, e.g.`app/importers/product_importer.rb`.

If the importer needs to do any complex processing or querying, consider extracting that logic into its own class as well. If your importer is crawling a site and parsing HTML into fields, both the crawler and HTML parser could be extracted to their own classes. These classes will be much easier to unit test on their own, and will limit the scope of dependencies they're exposed to.

### 2. Write Tests

This should go without saying: write automated tests. Importers often depend on logic and constraints in the database and deal with a variety of edge cases with source material. Most people aren't likely to manually test an importer once it's written, but they're likely candidates to break when changing their dependencies. At the bare minimum, write at least one integration test that goes from source material to the database testing the successful path.

When the source material is a file, create fixtures based on actual files supplied by the customer. However, trim down the number of records for the sake of performance. Replace any sensitive data (e.g. personally identifying customer information) with generated analogous [fake data](https://github.com/stympy/faker). Make sure to preserve the original formatting, including delimiters, line endings, and file encoding (CSV files from Excel often come with [`windows-1252`](http://en.wikipedia.org/wiki/Windows-1252) encoding).

If the source material is an API, use a gem like [vcr](https://github.com/vcr/vcr) to record the API requests on the first run, and playback the recordings subsequently. Your tests will be significantly faster with little consequence. You also gain the ability to work offline, and you won't risk hitting potential API rate limits.

### 3. Instrument and Log

> How long has the daily importer has been crashing? Did it always take this long to run? Did we get stale data last Thursday?

Every time your importer runs, gather statistics and log them to the database. Good values to capture are start and stop times, checksums of any input files, record counts,  the process id, and error messages. Store your import logs in the database, and your web application can provide a status message indicating how current the data is.

Logging doesn't have to end at the database. Even if you are rescuing exceptions to log them in the database, you should still reraise _unexpected_ exceptions to log them to your [favorite](http://raygun.io/) [error](https://www.honeybadger.io/) [tracker](https://airbrake.io/) [service](https://www.getsentry.com/).

### 4. Communicate Progress

> How long until it's finished? Is it frozen?

You don't want to start an import job and sit staring at the terminal wondering how it's doing. You need to be able to tell your team and your customer how long until the data is available. When running the importer interactively on the terminal, using a [progress bar library](https://github.com/jfelchner/ruby-progressbar) is an ideal solution that can additionally provide an ETA. As a worst case, take a page from every unit test framework on the planet: just print a '.' to stdout every time you've finished processing a record or file successfully. If your importer is running from a crontab, leverage the databased-backed logging from the previous section and regularly update the log with the current progress at an appropriate interval.

### 5. Be Idempotent

No importer ever gets run just once. Too often, developers will manually truncate their tables between test runs. This invariably leads to duplicate records when someone else runs the import again a year later. If your application treats your imported tables as read-only, it may suffice to use the easy strategy of automatically truncating before importing. 

However, if anything else modifies the table, you need to have a versioning strategy to merge the existing table data with data to import. A merge strategy may also be prefered for importers with a long run time that need the ability to stop and resume, or run incrementally. The simplest strategy is to establish updated_at timestamps on both the table and import source, and let the most recent overwrite the other. This much more complex than just clearing out the table. Make sure to discuss this consideration with your team and customer during initial development, and make sure you’re provided with the data to implement it.

### 6. Use Transactions

What happens if suddenly today's import data contains a value that makes your database throw an error half way through importing? If the importer isn't running within an **atomic** transaction, you'll potentially have a panicked early morning phone call. When you run into a surprise error, you need to be able to fail safely and roll back to the previous data.

You also don't want to expose your users to half-imported data while the importer is still actively running. We need **isolation** to ensure the new data isn't displayed until it's _all_ in place.

Relational databases have thankfully solved this problem with [ACID](http://en.wikipedia.org/wiki/ACID) transactions; you just need to remember to use them when appropriate. Using them from Ruby on Rails is as easy as a call to [`YourModel.transaction`](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/DatabaseStatements.html#method-i-transaction).

There are situations when you might not want to use a transaction. If the importer takes a long time to run and supports merging data between runs, you may want the ability to halt the import while it's running, keep the data in place, and resume it later.

### 7. Sanity Check Your Input

Eventually you're going to receive bad data to import. You don't want to poison your system with that bad data, especially with an automatic scheduled import. Some examples of bad data we've encountered:

- Missing files
- Existing, but blank files
- Files with only a subset of expected records
- Stale, previously imported files (track your file checksums!)
- Files with columns missing, added, or in a different order
- Files with different delimiters, line endings, and file encoding
- Sandbox/development data sent to the production server

Check for these potential issues both before the import starts and before the database transaction is committed; rollback if there's an issue. Creating custom exception classes for each condition will make your code more communicative, and provide better control over how to respond.

### 8. Leverage Your Database For Performance

It's natural for developers to write the entirety of the importer in the application layer. They'll read CSV's with the ruby [csv](http://ruby-doc.org/stdlib-2.1.1/libdoc/csv/rdoc/index.html) parser, and save each row off to the database with ActiveRecord:

{% highlight ruby %}
class ProductImporter
  def import(csv_path)
    log_import do
      Product.transaction do
        CSV.foreach csv_path do |row|
          product_for_csv_row(row).save!
        end
      end
    end
  end

  def product_for_csv_row(row)
    Product.new({
      name: row[0],
      # ...
    })
  end
end
{% endhighlight %}

Fundamentally, there's a lot to like about the above code. It's all implemented in the same layer and language, which makes it easy to reason about, easy to test, checks the data against your existing validations, and feels elegant. Unfortunately, there's an enormous amount of overhead with building and saving records one at a time through ActiveRecord.

Databases are made for speed. When importing large CSVs, move as much heavy lifting into the database layer as possible. In Postgres, use [`COPY`](http://www.postgresql.org/docs/9.3/static/sql-copy.html). In MySQL, use [`LOAD DATA`](http://dev.mysql.com/doc/refman/5.1/en/load-data.html)). I've reduced a importer's processing time from 15 minutes down to 3 seconds moving the processing from ActiveRecord directly into the database.

You may need to make some sacrifices in exchange for this performance boon. Data transformations between the CSV and the database now need to happen in SQL. You can achieve this by first importing to a separate [TEMPORARY](http://www.postgresql.org/docs/9.3/static/sql-createtable.html#AEN72686) table and transforming and loading via an `INSERT ... SELECT` statement. Obviously, you won't have application-layer tools available for doing this. We had to write [attr_encrypted_pgcrypto](https://github.com/gabetax/attr_encrypted_pgcrypto) because Postgres couldn't access [attr_encrypted](https://github.com/attr-encrypted/attr_encrypted)'s ruby-based OpenSSL implementation. You'll also lose validations implemented in your model classes, and may have to duplicate that logic using [database constraints](http://www.postgresql.org/docs/9.3/static/ddl-constraints.html) if it's important.

{% highlight ruby %}
class ProductImporter < Importer
  TEMP_TABLE    = 'products_import_temp'
  CSV_DELIMITER = ','
  CSV_COLUMNS   = %w(
    id
    name
    description
    price
  )

  def import(csv_path)
    log_import do
      create_temporary_table
      import_csv_to_temporary(csv_path)
      truncate_products
      transform_and_load_products
    end
  end

  def create_temporary_table
    execute \"CREATE TEMPORARY TABLE #{TEMP_TABLE} ( #{CSV_COLUMNS.map{|c| \"#{c} varchar(255) NOT NULL\"}.join(", ')})"
  end

  def import_csv_to_temporary(csv_path)
    execute <<-EOS
      COPY #{TEMP_TABLE}
      FROM '#{csv_path}'
      WITH
      DELIMITER AS '#{CSV_DELIMITER}'
      CSV FORCE NOT NULL #{CSV_COLUMNS.join(', ')}
    EOS
  end

  def truncate_products
    # `TRUNCATE` is not directly in ActiveRecord, and is not transaction-safe in MySQL
    Product.delete_all
  end

  def transform_and_load_products
    loan_insert = <<-EOS
      INSERT INTO products (created_at, updated_at, #{CSV_COLUMNS.join(', ')})
      (SELECT
        NOW() AS created_at,
        NOW() AS updated_at,
        id::INTEGER AS id,
        TRIM(name),
        TRIM(description),
        format_money(price)
        FROM #{TEMP_TABLE}
      )
    EOS
  end

  def format_money(field)
    "regexp_replace(#{field}, '[$,]', '', 'g')::DECIMAL(10,2) AS #{field}"
  end
end
{% endhighlight %}

## Wrapping Up

Applications can not live without their data, but often time imports don't get the  level of detail and forethought that they deserve. When you're designing a new importer, give forethought to its run time, how it will get run, how you'll monitor it, how reliable your data is, and the risks if the import fails. Let your context guide your implementation, and you'll have an importer you can depend on.
