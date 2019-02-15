---
layout: post
title: "An Object-oriented Moment"
date: 2019-02-15 13:12:00 -0500
excerpt: Leaning into object-oriented design to re-implement a procedural algorithm
categories: object-oriented refactoring
---

I had a rare epiphany not too long ago of how to reimplement a procedural
algorithm with what I believe to be a cleaner, object-oriented approach.  In
the past, I have often struggled to see these opportunities.  The first programs
that I ever wrote were not written in an object-oriented language, and I wonder
how much that has hindered my ability to design software in a steadfast OO
manner.  Nevertheless, given this latest discovery, I may have made a small
breakthrough.

## The Problem

At Stitch Fix, my team often finds ourselves in situations that call for
processing data in batches of a certain maximum size.  Sometimes our batch
sizes are dictated by third-party API limits and sometimes we need batching
to avoid long-running background jobs.  We, as an engineering organization,
consider it a best practice to avoid long-running background jobs.  Among other
benefits, this helps prevent lost work or large amounts of work that need to be
retried when background worker processes shut down unexpectedly.

## The System

Our email service provider (ESP) provides CSV files describing events for all
of our emails throughout the day.  These files can often be several gigabytes of
data, especially on days when we send out large promotional campaigns.  Every
day we have an internal service that downloads those files from our ESP and
saves the events to a database so that our customer experience team can see a
history of emails that a client has received.

The downloading and recording of events is done in Resque background jobs.  In
order to keep the background jobs speedy (ideally under 5 minutes of runtime),
we experimented and determined that we could save batches of 100,000 email
events at a time.

Alright, so, how do we create batches of 100,000 email events?  When we first
download the large events file from our ESP, we iterate through the file and
for every 100,000 rows in the CSV, we create a separate CSV file in an AWS S3
bucket.  Each background job saves the events in one of these smaller S3 files.

![Email events system diagram]({{"/images/email-events-diagram.png" | relative_url}}){: .centered }

## The Algorithm

### The Procedural Approach

So originally, I implemented a method to split large files and this first
attempt was extremely procedural.

```ruby
class FileSplitter
  BATCH_LIMIT = 100_000

  def split_large_csv_file(file_path)
    i = 0
    file_index = 1

    File.open(file_path, 'r') do |f|
      headers = f.gets

      csv = File.new("#{file_path}_#{file_index}", 'w')
      csv << headers

      while line = f.gets
        csv << line
        i += 1

        if i % BATCH_LIMIT == 0
          csv.close
          file_index += 1
          csv = File.new("#{file_path}_#{file_index}", 'w')
          csv << headers
        end
      end

      if i % BATCH_LIMIT != 0
        csv.close
      else
        file_index -= 1
      end
    end

    # Return an array of file paths representing the smaller CSVs
    (1..file_index).map { |i| "#{file_path}_#{i}" }
  end
end
```

Yikes.  I don't think there is anything wrong with procedural programming, but
in this case I could actually see the edge cases in the code; the implementation
looks like an off-by-one-error just waiting to happen.  To me, it felt
like the complexity of the algorithm was far too prevalent in the method.

My first thought was to break this method into smaller methods, but I
realized that would probably mean either creating more blocks to encapsulate
opening and closing files or sharing the responsibility between opening and
closing files between multiple methods.  I didn't like the sound of either of
those solutions.  In a rare moment of, I guess, inspiration (although that
sounds like I am giving myself too much credit), I paused to rethink my entire
approach.

It occurred to me to ask, how does Ruby, as a language, or perhaps more
accurately with its standard library, handle batching data?  Well, it turns out
that Ruby's Enumerable module provides the `each_slice` method.  Okay, so, all
I needed was to implement a class to represent one of those large CSV files and
this class needs to include the `Enumerable` module?  That sounded way too
simple, but it was easy enough to try out.

```ruby
class EventsCSV
  include Enumerable

  attr_reader :headers

  def initialize(file_path)
    @file_path = file_path
    @file = File.new(file_path, 'r')
    @headers = @file.gets
  end

  def close
    @file.close
  end

  def each
    while line = @file.gets
      yield line
    end
  end
end
```

Immediately, I started to see some benefits.  This EventsCSV class could also
encapsulate the concept of the headers in the CSV file.  We can hide that
complexity right in the constructor of the class; that works for me.

So using the `EventsCSV` class let's look at our `split_large_csv_file` method.

```ruby
class FileSplitter
  BATCH_LIMIT = 100_000

  def split_large_csv_file(file_path)
    events_csv = EventsCSV.new(file_path)
    file_index = 1

    events_csv.each_slice(BATCH_LIMIT) do |batch|
      File.open("#{file_path}_#{file_index}", 'w') do |csv|
        csv << events_csv.headers
        batch.each { |row| csv << row }
      end

      file_index += 1
    end

    # Return an array of file paths representing the smaller CSVs
    (1..file_index).map { |i| "#{file_path}_#{i}" }
  end
end
```

Okay, we are getting there.  There is some low-hanging fruit such that we can
rely on the Enumerable module even more.

```ruby
class FileSplitter
  BATCH_LIMIT = 100_000

  def split_large_csv_file(file_path)
    events_csv = EventsCSV.new(file_path)

    events_csv.each_slice(BATCH_LIMIT).each_with_index.map do |batch, file_index|
      split_file_path = "#{file_path}_#{file_index}"

      File.open(split_file_path, 'w') do |csv|
        csv << events_csv.headers
        batch.each { |row| csv << row }
      end

      split_file_path
    end
  end
end
```

One more, small change to extract the File IO into a separate method.

```ruby
class FileSplitter
  BATCH_LIMIT = 100_000

  def split_large_csv_file(file_path)
    events_csv = EventsCSV.new(file_path)

    events_csv.each_slice(BATCH_LIMIT).each_with_index.map do |batch, i|
      "#{file_path}_#{i}".tap do |split_file_path|
        write_batch_to_file(split_file_path, events_csv.headers, batch)
      end
    end
  end

  def write_batch_to_file(file_path, headers, batch)
    File.open(file_path, 'w') do |csv|
      csv << headers
      batch.each { |row| csv << row }
    end
  end
end
```

## Conclusions

As I said above, I was blown away by how a shift in approach could lead to a
drastically cleaner solution.  The OO approach in this case relies on the [Ruby
Enumerable module](https://ruby-doc.org/core-2.6.1/Enumerable.html) to deal with
the edge case complexities of batching data.  Therefore, we don't really have to
worry about testing those edge cases, either.  As long as we have implemented
`#each` correctly in our `EventsCSV` class, we can have high confidence that
`#each_slice` will also work correctly.

## Next Up

As nice as this solution worked out from a code standpoint, there were actually
some unforeseen memory issues when we released the code in production.  I share
how I debugged and ultimately resolved those issues
[here]({% post_url 2019-02-15-troubleshooting-ruby-enumerable-memory-usage %}).
