# memoize

###    Rationale

   Many computational operations take a significant amount of time and/or use
   an inordinate amount of resources. If subsequent calls to that function with
   the same parameters are guaranteed to return the same result, caching the
   result can lead to significant performance improvements. The process of
   caching such calls is called
   [memoization](http://en.wikipedia.org/wiki/Memoization).

###    Declaration

   Using memoization requires two simple steps: including the
   `Functional::Memo` module within a class or module and calling the `memoize`
   function to enable memoization on one or more methods.

   ```ruby
   Module EvenNumbers
     include Functional::Memoize

     self.first(n)
       (2..n).select{|i| i % 2 == 0 }
     end

     memoize :first
   end
   ```

   When a function is memoized an internal cache is created that maps arguments
   to return values. When the function is called the arguments are checked
   against the cache. If the args are found the method is not called and the
   cached result is returned instead.

###    Ramifications

   Memoizing long-running methods can lead to significant performance
   advantages. But there is a trade-off. Memoization may greatly increase the
   memory footprint of the application. The memo cache itself takes memory. The
   more arg/result pairs stored in the cache, the more memory is consumed.

#####    Cache Size Options

   To help control the size of the cache, a limit can be placed on the number
   of items retained in the cache. The `:at_most` option, when given, indicates
   the maximum size of the cache. Once the maximum cache size is reached, calls
   to to the method with uncached args will still result in the method being
   called, but the results will not be cached.

   ```ruby
   Module EvenNumbers
     include Functional::Memoize

     self.first(n)
       (2..n).select{|i| i % 2 == 0 }
     end

     memoize :first, at_most: 1000
   end
   ```

   There is no way to predict in advance what the proper cache size is, or if
   it should be restricted at all. Only performance testing under realistic
   conditions or profiling of a running system can provide guidance.

###    Restrictions

   Not all methods are good candidates for memoization.Only methods that are
   [idempotent](http://en.wikipedia.org/wiki/Idempotence), [referentially
   transparent](http://en.wikipedia.org/wiki/Referential_transparency_(computer_science)),
   and free of [side effects](http://en.wikipedia.org/wiki/Side_effect_(computer_science))
   can be effectively memoized. If a method creates side effects, such as
   writing to a log, only the first call to the method will create those side
   effects. Subsequent calls will return the cached value without calling the
   method.

   Similarly, methods which change internal state will only update the state on
   the initial call. Later calls will not result in state changes, they will
   only return the original result. Subsequently, instance methods cannot be
   memoized. Objects are, by definition, stateful. Method calls exist for the
   purpose of changing or using the internal state of the object. Such methods
   cannot be effectively memoized; it would require the internal state of the
   object to be cached and checked as well.

   Block parameters pose a similar problem. Block parameters are inherently
   stateful (they are closures which capture the enclosing context). And there
   is no way to check the state of the block along with the args to determine
   if the cached value should be used. Subsequently, and method call which
   includes a block will result in the cache being completely skipped. The base
   method will be called and the result will not be cached. This behavior will
   occur even when the given method was not programmed to accept a block
   parameter. Ruby will capture any block passed to any method and make it
   available to the method even when not documented as a formal parameter or
   used in the method. This has the interesting side effect of allowing the
   memo cache to be skipped on any method call, simply be passing a block
   parameter.

   ```ruby
   EvenNumbers.first(100)         causes the result to be cached
   EvenNumbers.first(100)         retrieves the previous result from the cache
   EvenNumbers.first(100){ nil }  skips the memo cache and calls the method again
   ```

###   Complete Example

   The following example is borrowed from the book [Functional Thinking](http://shop.oreilly.com/product/0636920029687.do)
   by Neal Ford. In his book he shows an example of memoization in Groovy by
   summing factors of a given number. This is a great example because it
   exhibits all the criteria that make a method a good memoization candidate:

   * Idempotence
   * Referential transparency
   * Stateless
   * Free of side effect
   * Computationally expensive (for large numbers)

   The following code implements Ford's algorithms in Ruby, then memoizes two
   key methods. The Ruby code:

   ```ruby
   require 'functional'

   class Factors
     include Functional::Memo

     def self.sum_of(number)
       of(number).reduce(:+)
     end

     def self.of(number)
       (1..number).select {|i| factor?(number, i)}
     end

     def self.factor?(number, potential)
       number % potential == 0
     end

     def self.perfect?(number)
       sum_of(number) == 2 * number
     end

     def self.abundant?(number)
       sum_of(number) > 2 * number
     end

     def self.deficient?(number)
       sum_of(number) < 2 * number
     end

     memoize(:sum_of)
     memoize(:of)
   end
   ```

   This code was tested in IRB using MRI 2.1.2 on a MacBook Pro. The `sum_of`
   method was called three times against the number 10,000,000 and the
   benchmark results of each run were captured. The test code:

   ```ruby
   require 'benchmark'

   3.times do
     stats = Benchmark.measure do
       Factors.sum_of(10_000_000)
     end
     puts stats
   end
   ```

   The results of the benchmarking are very revealing. The first run took over
   a second to calculate the results. The two subsequent runs, which retrieved
   the previous result from the memo cache, were nearly instantaneous:

   ```
   1.080000   0.000000   1.080000 (  1.077524)
   0.000000   0.000000   0.000000 (  0.000033)
   0.000000   0.000000   0.000000 (  0.000008)
   ```

   The same code run on the same computer using JRuby 1.7.12 exhibited similar
   results:

   ```
   1.800000   0.030000   1.830000 (  1.494000)
   0.000000   0.000000   0.000000 (  0.000000)
   0.000000   0.000000   0.000000 (  0.000000)
   ```

###    Inspiration

   * [Memoization](http://en.wikipedia.org/wiki/Memoization) at Wikipedia
   * Clojure [memoize](http://clojuredocs.org/clojure_core/clojure.core/memoize) function
