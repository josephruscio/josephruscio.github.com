---
title: Aggregate statistics gem released!
---

Finally got around to putting together a decent README (and a few last bugfixes)
for my Aggregate statistics gem. With that out of the way, its ready for
public consumption. Aggregate is an easy-to-use ruby implementation of a
statistics aggregator that includes configurable histogram (binary or linear)
support. Stats are aggregated in a _streaming_ fashion that doesn't record/store
any of the individual sample values, enabling statistics tracking across a
practically unlimited number of samples without any impact on your application's
performance or memory footprint.

Perhaps more importantly than the basic aggregate statistics (e.g. mean,
standard deviation, max, min, etc) Aggregate also maintains a histogram of the
sample distribution. For anything other than normally distributed data averages
are insufficient at best and often downright misleading. 37Signals recently
posted a
[terse but effective explanation](http://37signals.com/svn/posts/1836-the-problem-with-averages)
on the importance of histograms. Here's a second more
[detailed posting](http://www.zedshaw.com/essays/programmer_stats.html) by Zed
Shaw authored in his inimitable and entertaining style.

Full source code and a detailed README can be found on the
[project homepage](http://github.com/josephruscio/aggregate) and the gem is
available on [gemcutter.org](http://www.gemcutter.org/gems/aggregate).
But just to whet your appetite, here's a quick example:

    require 'rubygems'
    require 'aggregate'

    # Create an Aggregate instance
    binary_aggregate = Aggregate.new
    linear_aggregate = Aggregate.new(0, 65536, 8192)

    65536.times do
      x = rand(65536)
      binary_aggregate >> x
      linear_aggregate >> x
    end

    puts "\n** Binary Histogram **\n%s" % [binary_aggregate.to_s]
    puts "\n** Linear Histogram **\n%s" % [linear_aggregate.to_s]

And the resulting output:

    macbook:aggregate(master) jruscio$ ruby ./aggregate_example.rb 

    ** Binary Histogram **
    value |------------------------------------------------------------------| count
        1 |                                                                  |     1
        2 |                                                                  |     1
        4 |                                                                  |     2
        8 |                                                                  |    10
       16 |                                                                  |    24
       32 |                                                                  |    33
       64 |                                                                  |    71
      128 |                                                                  |   121
      256 |                                                                  |   258
      512 |@                                                                 |   497
     1024 |@@                                                                |  1061
     2048 |@@@@                                                              |  2004
     4096 |@@@@@@@@                                                          |  4186
     8192 |@@@@@@@@@@@@@@@@                                                  |  8257
    16384 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                                 | 16345
    32768 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@| 32665
	  ~
    Total |------------------------------------------------------------------| 65536

    ** Linear Histogram **
    value |------------------------------------------------------------------| count
        0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ |  8269
     8192 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ |  8257
    16384 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ |  8302
    24576 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   |  8043
    32768 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ |  8273
    40960 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   |  8052
    49152 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|  8315
    57344 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   |  8025
    Total |------------------------------------------------------------------| 65536

Feedback welcome and enjoy!
