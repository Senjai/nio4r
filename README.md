New IO for Ruby
===============
[![Build Status](http://travis-ci.org/tarcieri/nio4r.png)](http://travis-ci.org/tarcieri/nio4r)

When it comes to managing many IO objects on Ruby, there aren't a whole lot of
options. The most powerful API Ruby itself gives you is Kernel.select, and
select is hurting when it comes to performance and in terms of having a nice
API.

Once upon a time Java was a similar mess. They got out of it by adding the
Java NIO API. Java NIO provides a high performance selector API for monitoring
large numbers of file descriptors.

This library aims to incorporate the ideas of Java NIO in Ruby. These are:

* Expose high level interfaces for doing high performance IO, but keep the
  codebase small to encourage multiple implementations on different platforms
* Be as portable as possible, in this case across several Ruby VMs
* Provide inherently thread-safe facilities for working with IO objects

Supported Platforms
-------------------

nio4r is known to work on the following Ruby implementations:

* MRI/YARV 1.8.7, 1.9.2, 1.9.3
* REE (2011.12)
* JRuby 1.6.x (and likely earlier versions too)
* Rubinius 1.x/2.0
* A pure Ruby implementation based on Kernel.select is also provided

Platform notes:

* MRI/YARV and Rubinius implement nio4j with a C extension based on libev,
  which provides a high performance binding to native IO APIs
* JRuby uses a Java extension based on the high performance Java NIO subsystem
* A pure Ruby implementation is also provided for Ruby implementations which
  don't implement the MRI C extension API

Usage
-----

### Selectors

The NIO::Selector class is the main API provided by nio4r. Use it where you
might otherwise use Kernel.select, but want to monitor the same set of IO
objects across multiple select calls rather than having to reregister them
every single time:

```ruby
require 'nio'

selector = NIO::Selector.new
```

To monitor IO objects, attach them to the selector with the NIO::Selector#register
method, monitoring them for read readiness with the :r parameter, write
readiness with the :w parameter, or both with :rw.

```ruby
>> reader, writer = IO.pipe
 => [#<IO:0xf30>, #<IO:0xf34>]
>> monitor = selector.register(reader, :r)
 => #<NIO::Monitor:0xfbc>
```

After registering an IO object with the selector, you'll get a NIO::Monitor
object which you can use for managing how a particular IO object is being
monitored. Monitors will store an arbitrary value of your choice, which
provides an easy way to implement callbacks:

```ruby
>> monitor = selector.register(reader, :r)
 => #<NIO::Monitor:0xfbc>
>> monitor.value = proc { puts "Got some data: #{monitor.io.read_nonblock(4096)}" }
 => #<Proc:0x1000@(irb):4>
```

The main method of importance is NIO::Selector#select, which monitors all
registered IO objects and returns an array of monitors that are ready.

```ruby
>> writer << "Hi there!"
 => #<IO:0x103c>
>> ready = selector.select
 => [#<NIO::Monitor:0xfbc>]
>> ready.each { |m| m.value.call }
Got some data: Hi there!
 => [#<NIO::Monitor:0xfbc>]
```

By default, NIO::Selector#select will block indefinitely until one of the IO
objects being monitored becomes ready. However, you can also pass a timeout to
wait in seconds to NIO::Selector#select just like you can with Kernel.select:

```ruby
ready = selector.select(15) # Wait 15 seconds
```

If a timeout occurs, ready will be nil.

You can avoid allocating an array each time you call NIO::Selector#select by
passing a block to select. The block will be called for each ready monitor
object, with that object passed as an argument. The number of ready monitors
is returned as a Fixnum:

```ruby
>> selector.select { |m| m.value.call }
Got some data: Hi there!
 => 1
```

When you're done monitoring a particular IO object, just deregister it from
the selector:

```ruby
selector.deregister(reader)
```

### Monitors

Monitors provide methods which let you introspect on why a particular IO
object was selected. These methods are not thread safe unless you are holding
the selector lock (i.e. if you're in a block pased to #select). Only use them
if you aren't concerned with thread safety, or you're within a #select
block:

- ***#interests***: what this monitor is interested in (:r, :w, or :rw)
- ***#readiness***: what the monitored IO object is ready for according to the last select operation
- ***#readable?***: was the IO readable last time it was selected?
- ***#writable?***: was the IO writable last time it was selected?

Monitors also support a ***#value*** and ***#value=*** method for storing a
handle to an arbitrary object of your choice (e.g. a proc)

### Buffers

NIO::Buffer is a fast byte queue which is primarily intended for non-blocking
I/O applications but is suitable wherever buffering is required.

NIO::Buffer provides a subset of the methods available in Strings:

```ruby
>> buf = NIO::Buffer.new
=> #<NIO::Buffer:0x12fc708>
>> buf << "foo"
=> "foo"
>> buf << "bar"
=> "bar"
>> buf.to_str
=> "foobar"
>> buf.size
=> 6
```

The NIO::Buffer#<< method is an alias for NIO::Buffer#append.  A prepend method
is also available:

```ruby
>> buf = NIO::Buffer.new
=> #<NIO::Buffer:0x12f5250>
>> buf.append "bar"
=> "bar"
>> buf.prepend "foo"
=> "foo"
>> buf.append "baz"
=> "baz"
>> buf.to_str
=> "foobarbaz"
```

NIO::Buffer#read can be used to retrieve the contents of a buffer.  You can mix
reads alongside adding more data to the buffer:

```ruby
>> buf = NIO::Buffer.new
=> #<NIO::Buffer:0x12fc5f0>
>> buf << "foo"
=> "foo"
>> buf.read 2
=> "fo"
>> buf << "bar"
=> "bar"
>> buf.read 2
=> "ob"
>> buf << "baz"
=> "baz"
>> buf.read 3
=> "arb"
```

Finally, NIO::Buffer provides methods for performing non-blocking I/O with the
contents of the buffer. The NIO::Buffer#read_from(IO) method will read as much
data as possible from the given IO object until the read would block.

The NIO::Buffer#write_to(IO) method writes the contents of the buffer to the
given IO object until either the buffer is empty or the write would block:

```ruby
>> buf = NIO::Buffer.new
=> #<NIO::Buffer:0x12fc5c8>
>> file = File.open("README")
=> #<File:README>
>> buf.read_from(file)
=> 1713
>> buf.to_str
=> "= NIO::Buffer\n\nNIO::Buffer is a fast byte queue...
```

If the file descriptor is not ready for I/O, the Errno::EAGAIN exception is
raised indicating no I/O was performed.

Concurrency
-----------

nio4r provides internal locking to ensure that it's safe to use from multiple
concurrent threads. Only one thread can select on a NIO::Selector at a given
time, and while a thread is selecting other threads are blocked from
registering or deregistering IO objects. Once a pending select has completed,
requests to register/unregister IO objects will be processed.

NIO::Selector#wakeup allows one thread to unblock another thread that's in the
middle of an NIO::Selector#select operation. This lets other threads that need
to communicate immediately with the selector unblock it so it can process
other events that it's not presently selecting on.

What nio4r is not
-----------------

nio4r is not a full-featured event framework like EventMachine or Cool.io.
Instead, nio4r is the sort of thing you might write a library like that on
top of. nio4r provides a minimal API such that individual Ruby implementers
may choose to produce optimized versions for their platform, without having
to maintain a large codebase.

As of the time of writing, the current implementation is (approximately):

* 200 lines of Ruby code
* 700 lines of "custom" C code (not counting libev)
* 400 lines of Java code

nio4r is also not a replacement for Kinder Gentler IO (KGIO), a set of
advanced Ruby IO APIs. At some point in the future nio4r might provide a
cross-platform implementation that uses KGIO on CRubies, and Java NIO on JRuby,
however this is not the case today.

License
-------

Copyright (c) 2011-12 Tony Arcieri. Distributed under the MIT License. See
LICENSE.txt for further details.

Includes libev. Copyright (C)2007-09 Marc Alexander Lehmann. Distributed under
the BSD license. See ext/libev/LICENSE for details.
