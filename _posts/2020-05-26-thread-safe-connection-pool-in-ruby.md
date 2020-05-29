---
layout: post
title:  "Thread Safe Connection Pool in Ruby"
date:   2020-05-26 10:00:00 -0700
categories: rails ruby connection_pool threads
---
To demonstrate a thread safe implementation of a connection pool, we will use a class instance variable, a Mutex, and the [connection_pool](https://github.com/mperham/connection_pool){:target="_blank"} gem.

### Class Instance Variables

Class instance variables work like regular class variables except for two main differences.

1. Class instance variables are available only to class methods and not to instance methods.
2. Class instance variables are not shared with sub-classes. They belong exclusively to the class itself.

We will use class instance variables (mainly for reason 1) to share a connection pool which all threads will be able to access. Here is a quick example of what class instance variables can do for us.

```ruby
class Example
  @my_value = 1

  class << self
    # class methods

    def my_value
      @my_value
    end

    def my_value=(value)
      @my_value = value
    end
  end

  # instance methods

  def my_value
    @my_value
  end

  def my_value=(value)
    @my_value = value
  end
end

# class method
puts Example.my_value # my_value = 1
Example.my_value = 2
puts Example.my_value # my_value = 2

# instance method
a = Example.new
puts a.my_value # my_value is not initialized at the instance level
a.my_value = 3
puts a.my_value # my_value = 3

# class method
puts Example.my_value # my_value is still 2 at the class level
```

When `my_value` was accessed by class methods, its value was only accessed and updated at the class level. When accessed as an instance method, it was accessed and updated in a separate memory space at the instance level.

If you want to learn more about class instance variables, take a look at this [article](https://www.codegram.com/blog/understanding-class-instance-variables-in-ruby/){:target="_blank"} and this [post](https://stackoverflow.com/questions/15773552/ruby-class-instance-variable-vs-class-variable){:target="_blank"}.

Now that we have a way to access the *same* connection pool across threads, we need to actually create the pool so threads can use it.

### Connection Pool

The [connection_pool](https://github.com/mperham/connection_pool){:target="_blank"} gem will allow us to create a pool of connections which we can use to grab a connection and return it after we're done using it. This allows us to specify a static number of connections and forces our threads to share those connections.

```ruby
class ConnectionManager
  @pool = nil

  class << self
    def connection_pool
      @pool || create_pool
    end

    private

    def create_pool
      @pool = ConnectionPool.new(size: 10) do
        Dalli::Client.new
      end
    end
  end
end
```

With this, we can now access the shared connection pool across threads by calling `ConnectionManager.connection_pool`. To use one of the connections in the pool, we use `with` which will yield a connection.

```ruby
ConnectionManager.connection_pool.with do |conn|
  # do work with the connection
  conn.get(...)
end
```

However, there is a slight problem. Multiple threads could access `connection_pool` at the same time, see that `@pool` hasn't been initialized, and then each one of those threads would create another connection pool. This would cause us to create extra pools (and extra connections) which we don’t want. We only want the first thread that accesses `connection_pool` to initialize `@pool` and not the consequent ones.

To demonstrate the race condition, we will slightly modify the class above to show how that’s possible.

```ruby
class ConnectionManager
  @my_var = nil

  class << self
    def connection_pool
      @my_var || create_pool
    end

    private

    def create_pool
      return @my_var if @my_var
      # if this was thread safe, @my_var would only be updated once
      puts "updating my_var"
      @my_var = 1
    end
  end
end

threads = []
10.times {
  threads << Thread.new {
    puts "[ThreadId=#{Thread.current.object_id}] my_values=#{ConnectionManager.connection_pool}"
  }
}
threads.each(&:join)
```

**Output**

```ruby
updating my_var
[ThreadId=70347336477020] my_values=1
updating my_var
[ThreadId=70347336475740] my_values=1
[ThreadId=70347315666620] my_values=1
[ThreadId=70347315666480] my_values=1
[ThreadId=70347315666340] my_values=1
[ThreadId=70347315666200] my_values=1
[ThreadId=70347315666060] my_values=1
[ThreadId=70347336475320] my_values=1
updating my_var
[ThreadId=70347336476360] my_values=1
updating my_var
[ThreadId=70347315665900] my_values=1
```

As you can see, `create_pool` ends up being accessed multiple times, even when we have `return @my_var if @my_var` to check for initialization. There is a race condition between threads that ends up updating `@my_var` multiple times. We only want `@my_var` to be updated once no matter how many times `create_pool` is called.

We can use a `Mutex` to solve this.

### Mutex

In the ruby documentation, the `Mutex` class has the following description

> Mutex implements a simple semaphore that can be used to coordinate access to shared data from multiple concurrent threads.¹

We can use a mutex to avoid creating multiple pools when threads initially access `ConnectionManager.connection_pool`.

```ruby
class ConnectionManager
  @pool = nil
  @mutex = Mutex.new

  class << self
    def connection_pool
      @pool || create_pool
    end

    private

    def create_pool
      # allows only one thread to access the mutex at a time which
      # avoids creating multiple pools
      @mutex.synchronize do
        return @pool if @pool # in case threads after the first enter the mutex
        @pool = ConnectionPool.new(size: 10) do
          Dalli::Client.new
        end
      end
    end
  end
end
```

We’ve introduced a mutex in `create_pool` to only allow one thread to create the pool at a time. Additionally, if threads are waiting for the mutex to be free while the pool is initially being created, `return @pool if @pool` will stop those threads from creating a new pool when they get a chance to access the mutex.

At this point, we should be able to use `connection_pool` safely across our threads.

### In Action

So we can see this in action, we will again slightly modify the above class so we can run it in many threads and see what the output is.

### With a Mutex

```ruby
class ConnectionManager
  @my_var = nil
  @mutex = Mutex.new

  class << self
    def connection_pool
      @my_var || create_pool
    end

     private

    def create_pool
      @mutex.synchronize do
        return @my_var if @my_var
        puts "updating my_var"
        @my_var = 1
      end
    end
  end
end

threads = []
10.times {
  threads << Thread.new {
    puts "[ThreadId=#{Thread.current.object_id}] my_values=#{ConnectionManager.connection_pool}"
  }
}
threads.each(&:join)
```

**Output**

```ruby
updating my_var
[ThreadId=70279845930440] my_values=1
[ThreadId=70279850352420] my_values=1
[ThreadId=70279850352280] my_values=1
[ThreadId=70279850352140] my_values=1
[ThreadId=70279850352000] my_values=1
[ThreadId=70279850351840] my_values=1
[ThreadId=70279850351700] my_values=1
[ThreadId=70279850351540] my_values=1
[ThreadId=70279845930020] my_values=1
[ThreadId=70279845929580] my_values=1
```

Now, `@my_var` is only initialized once by the first thread and returned in consequent calls and threads.

There you have it! How to a create thread safe connection pool using class instance variables, a Mutex, and the [connection_pool](https://github.com/mperham/connection_pool){:target="_blank"} gem.

### References

1. [https://ruby-doc.org/core-2.6/Mutex.html](https://ruby-doc.org/core-2.6/Mutex.html){:target="_blank"}