---
layout: post
title: "Troubleshooting Ruby Enumerable Memory Usage"
date: 2019-02-15 14:32:00 -0500
excerpt: Analyzing Ruby memory usage related to the Enumerable module's `#each_slice` method
categories: memory performance debugging
---

Previously, I [discussed]({% post_url 2019-02-15-an-object-oriented-moment %})
an implementation within an ETL system where I removed a lot of complexity from
my code by relying on Ruby's
[Enumerable](https://ruby-doc.org/core-2.6.1/Enumerable.html) module.  While
using the default implementation of `Enumerable#each_slice` is simplest from a
readability and maintenance standpoint, I found that the `#each_slice` method
can use a significant amount of memory under certain conditions.  In this post,
I will walk through my debugging process and show a solution to the memory
issues that I discovered.  By overriding the `#each_slice` method in my class,
I was able to optimize for memory conservation.

## The Enumerable Approach

To provide context for this investigation, I have two classes.
  * `EventsCSV` - represents a large CSV of records.
  * `FileSplitter` - splits the large CSV into smaller, more manageable files.

In practice, the smaller files are uploaded to a remote file repository and are
each consumed in parallel via background jobs; those details are not discussed
in greater detail here.

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

class FileSplitter
  BATCH_LIMIT = 100_000

  def split_large_csv_file(file_path)
    events_csv = EventsCSV.new(file_path)

    events_csv
      .each_slice(BATCH_LIMIT)
      .each_with_index
      .map do |batch, i|
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

So, in production, the CSV files that are processed can contain multiple
gigabytes of data.  Shortly after releasing this code, I observed that the
worker processes splitting large CSV files were running out of memory.  The
result was failed Resque jobs due to `Resque::DirtyExit` errors.

## Debugging

Initially, I assumed that I was not releasing the memory from each separate
slice of data as I iterated through the CSV file in batches.  In order to try to
reveal the leak, I included the very helpful
[`get_process_mem` gem](https://github.com/schneems/get_process_mem).

After pulling the `EventsCSV` and `FileSplitter` into a easily executable,
standalone program, I began introspecting the code's memory usage.  Using
a CSV with around 4 million records, I decided to execute the following script.

```ruby
# profile.rb

require 'get_process_mem'
require 'time'

require_relative 'events_csv'
require_relative 'file_splitter'

start = Time.now
file_path = 'big_csv.csv'
file_splitter = FileSplitter.new
split_files = file_splitter
              .split_large_csv_file(file_path)

puts "Finished in #{(Time.now - start).round(3)} seconds"
```

Even though my main goal was to decrease the memory usage of my algorithm, I
also wanted to be aware of any significant performance penalties as the code
consumed less memory.  Therefore, I decided to print the execution time of the
script.

So, using the [`get_process_mem` gem](https://github.com/schneems/get_process_mem),
I was able to log the memory usage of the Ruby script like so:

```ruby
class FileSplitter
  BATCH_LIMIT = 100_000

  def split_large_csv_file(file_path)
    events_csv = EventsCSV.new(file_path)
    print_memory_usage("BEFORE each_slice")

    events_csv
      .each_slice(BATCH_LIMIT)
      .each_with_index
      .map do |batch, i|
      print_memory_usage("BEFORE batch #{i}")

      "#{file_path}_#{i}".tap do |split_file_path|
        write_batch_to_file(split_file_path, events_csv.headers, batch)
        print_memory_usage("AFTER batch #{i}")
      end
    end
  end

  def write_batch_to_file(file_path, headers, batch)
    File.open(file_path, 'w') do |csv|
      csv << headers
      batch.each { |row| csv << row }
    end
  end

  def print_memory_usage(prefix = '')
    mb = GetProcessMem.new.mb
    puts "#{prefix} - MEMORY USAGE(MB): #{mb.round}"
  end
end
```

Alright, so running `ruby profile.rb` produced the following output in the
console:

```
BEFORE each_slice - MEMORY USAGE(MB): 17
BEFORE batch 0 - MEMORY USAGE(MB): 38
AFTER batch 0 - MEMORY USAGE(MB): 41
BEFORE batch 1 - MEMORY USAGE(MB): 60
AFTER batch 1 - MEMORY USAGE(MB): 64
BEFORE batch 2 - MEMORY USAGE(MB): 82
AFTER batch 2 - MEMORY USAGE(MB): 86
BEFORE batch 3 - MEMORY USAGE(MB): 106
AFTER batch 3 - MEMORY USAGE(MB): 106
BEFORE batch 4 - MEMORY USAGE(MB): 109
AFTER batch 4 - MEMORY USAGE(MB): 109
BEFORE batch 5 - MEMORY USAGE(MB): 110
AFTER batch 5 - MEMORY USAGE(MB): 110
BEFORE batch 6 - MEMORY USAGE(MB): 111
AFTER batch 6 - MEMORY USAGE(MB): 111
BEFORE batch 7 - MEMORY USAGE(MB): 127
AFTER batch 7 - MEMORY USAGE(MB): 130
BEFORE batch 8 - MEMORY USAGE(MB): 151
AFTER batch 8 - MEMORY USAGE(MB): 151
BEFORE batch 9 - MEMORY USAGE(MB): 161
AFTER batch 9 - MEMORY USAGE(MB): 152
...
BEFORE batch 40 - MEMORY USAGE(MB): 158
AFTER batch 40 - MEMORY USAGE(MB): 158
BEFORE batch 41 - MEMORY USAGE(MB): 166
AFTER batch 41 - MEMORY USAGE(MB): 166
BEFORE batch 42 - MEMORY USAGE(MB): 168
AFTER batch 42 - MEMORY USAGE(MB): 168
Finished in 6.361 seconds
```

In production, my containers have 512 MB of available memory, so given that
this barebones script consumes 168 MB of memory,  it's reasonable that a
container that also runs a full Rails app could exhaust its available resources.
Also, notice that the memory usage jumps significantly after we start calling
`#each_slice`.  Furthermore, the memory usage eventually plateaus so I felt it
safe to assume, at this point, that there wasn't a memory leak in the code; the
implementation did not appear to be retaining references to each separate slice
of data.  If that was the problem, I would have expected memory usage to keep
climbing indefinitely at a more-or-less linear manner.  To verify that
`#each_slice` was using a lot of memory, I tried reducing the `BATCH_LIMIT`
from 100,000 to 10,000.

```
BEFORE each_slice - MEMORY USAGE(MB): 17
BEFORE batch 0 - MEMORY USAGE(MB): 19
AFTER batch 0 - MEMORY USAGE(MB): 19
BEFORE batch 1 - MEMORY USAGE(MB): 21
AFTER batch 1 - MEMORY USAGE(MB): 21
BEFORE batch 2 - MEMORY USAGE(MB): 23
AFTER batch 2 - MEMORY USAGE(MB): 23
BEFORE batch 3 - MEMORY USAGE(MB): 25
AFTER batch 3 - MEMORY USAGE(MB): 25
BEFORE batch 4 - MEMORY USAGE(MB): 26
AFTER batch 4 - MEMORY USAGE(MB): 26
...
BEFORE batch 426 - MEMORY USAGE(MB): 40
AFTER batch 426 - MEMORY USAGE(MB): 40
BEFORE batch 427 - MEMORY USAGE(MB): 40
AFTER batch 427 - MEMORY USAGE(MB): 40
BEFORE batch 428 - MEMORY USAGE(MB): 40
AFTER batch 428 - MEMORY USAGE(MB): 40
Finished in 10.934 seconds
```

So when using smaller batches, the programs consumes less memory.  To me, that
indicated that the implementation of `Enumerable#each_slice` consumes memory
according to the size of the slices.  That's not entirely surprising, but
then I started to wonder if there was an alternative `#each_slice`
implementation that would consume less memory with large batches.

## The Fix

So I wanted to try overriding `Enumerable#each_slice` with my own
implementation.  But what exactly do I want to return as a slice?  Well, I
knew how I was going to use my `#each_slice` implementation in practice; I knew
that, in my `FileSplitter` class, I was going to call:

```ruby
events_csv
  .each_slice(BATCH_LIMIT)
  .each_with_index
  .map do |batch, i|
  ...
end
```

So, it looks like whatever I return from `#each_slice` should also include
`Enumerable` so that `#each_with_index` and `#map` will work as expected.

```ruby
class EventsCSV
  def each_slice(n)
    # Something Enumerable
  end
end
```

So I went ahead and just created a sort of placeholder class just to wrap my
head around how this was going to work.  I figured that this class would at
least require a reference to the `EventsCSV` and to know the slice size in
order to work.

```ruby
class EventsCSV::SliceEnumerator
  include Enumerable

  def initialize(events_csv, slice_size)
    @events_csv = events_csv
    @slice_size = slice_size
  end

  def each
    # yields a slice, I guess?
  end
end
```

Okay, so using this model, what is a slice exactly?  I knew that I did not want
to build an array of the given slice size; that is why `Enumerable#each_slice`
consumes so much memory.  Rather, I wanted to yield one line of the file at
a time and to stop after reading a given number of lines.  So something like:

```ruby
class EventsCSV::SliceEnumerator
  include Enumerable

  def initialize(events_csv, slice_size)
    @events_csv = events_csv
    @slice_size = slice_size
  end

  def each
    yield Slice.new(@events_csv, @slice_size) while @events_csv.any?
  end
end

class EventsCSV::Slice
  include Enumerable

  def initialize(events_csv, size)
    @events_csv = events_csv
    @size = size
  end

  def each
    @events_csv.each_with_index do |row, i|
      yield row
      break if i + 1 == @size
    end
  end
end
```

A small aside, I also had to implement `EventsCSV#any?`.  Because the
`EventsCSV#each` calls `@file.gets`, calling `Enumerable#any?` will actually
read a line from the file; `Enumerable#any?` proved to be destructive.  So
we also have:

```ruby
class EventsCSV
  # EventsCSV stuff...

  def any?
    !@file.eof?
  end
end
```

So after setting the `BATCH_LIMIT` back to 100,000, the results of the
profiling script were:

```
BEFORE each_slice - MEMORY USAGE(MB): 17
BEFORE batch 0 - MEMORY USAGE(MB): 17
AFTER batch 0 - MEMORY USAGE(MB): 19
BEFORE batch 1 - MEMORY USAGE(MB): 19
AFTER batch 1 - MEMORY USAGE(MB): 20
BEFORE batch 2 - MEMORY USAGE(MB): 20
AFTER batch 2 - MEMORY USAGE(MB): 20
...
BEFORE batch 40 - MEMORY USAGE(MB): 20
AFTER batch 40 - MEMORY USAGE(MB): 20
BEFORE batch 41 - MEMORY USAGE(MB): 20
AFTER batch 41 - MEMORY USAGE(MB): 20
BEFORE batch 42 - MEMORY USAGE(MB): 20
AFTER batch 42 - MEMORY USAGE(MB): 20
Finished in 4.652 seconds
```

Perfect!  That's exactly what I was looking to accomplish; now I could process
large batches without consuming very much memory.
