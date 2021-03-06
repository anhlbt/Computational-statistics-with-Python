
Distributed computing for Big Data
==================================

-  many (cheapish) machines (known as nodes)
-  loosely coupled
-  fault tolerant

Why and when does distributed computing matter?
-----------------------------------------------

-  data growing exponentially

   -  Whole genome sequencing 100-200 GB per BAM file
   -  NYSE 1 TB of trade data per day
   -  Large Hadron Collider 15 PB per year (1 PB = 1,000 TB)

-  storage :math:`\gg` access speeds

   -  how long does it take to read or write a 1 TB disk?

      -  parallel reads can result in large speed ups

-  most relevant for **big** data

   -  1 billion rows
   -  :math:`\gg` 128 MB (default block size for Hadoop)

-  other solutions may be more suitable

   -  shared memory parallel system
   -  Relational databases (seek time is bottleneck)
   -  Grid computing for compute-intensive jobs where netwrok bandwidth
      is not bottleneck (HPC, MPI)
   -  Volunteer computing (compute time :math:`\gg` data transfer time)

Ingredients for effiicient distributed computing
------------------------------------------------

-  Problems

   -  high likelihood of hardware failure
   -  combine data from nodes in cluster

-  Hadoop

   -  Distributed file storage (HDFS)
   -  Cluster resource mangagement (YARN)
   -  Analysis model (MapReduce, Spark, Impala)

-  Programming style

   -  Functional
   -  Declarative

What is Hadoop?
---------------

Hadoop is a framework for distributed programming that handles failures
transparently and provides a way to robuslty code programs for execution
on a cluster. The main modules are

-  A distributed file system (HDFS - Hadoop Distributed File System)
-  A cluster manager (YARN - Yet Anther Resource Negotiator)
-  A parallel programming model for large data sets (MapReduce)

There is also an ecosystem of tools with very whimsical names built upon
the Hadoop framework, and this ecosystem can be bewildering. We will
minly look at distributed compuitng alternatives to MapReduce that can
run on HDFS - spefically ``Spark`` and ``Impala``. Also of interest is
``Mahout``, a parallel machine learing library built on top of
``MapReduce`` and ``spark``.

See the `official documnetation
here <http://hadoop.apache.org/docs/current/index.html>`__

Installation
~~~~~~~~~~~~

The simplest way to try out the Hadoop system is probbaly to install the
`Cloudera Virtual Machine
image <http://www.cloudera.com/content/cloudera/en/downloads/quickstart_vms/cdh-5-3-x.html>`__
or to use `Amazon Elastic
MapRedcue <http://aws.amazon.com/elasticmapreduce/>`__. If you install
`from
scratch <https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html>`__,
there are some confiugration steps to overcome. The following example
assumes that Hadoop has been installed locally and the path to Hadoop
executables has been exported.

Review of functional programming
--------------------------------

.. code:: python

    lambda
    map
    filter
    reduce
    fold
    concat
    flatmap
    aggregate
    groupby

.. code:: python

    from __future__ import division
    import os
    import sys
    import glob
    import matplotlib.pyplot as plt
    import numpy as np
    import pandas as pd
    %matplotlib inline
    %precision 4
    plt.style.use('ggplot')


.. code:: python

    ! pip install toolz


.. parsed-literal::

    Requirement already satisfied (use --upgrade to upgrade): toolz in /Users/cliburn/anaconda/lib/python2.7/site-packages


.. code:: python

    from toolz import countby, groupby, accumulate, reduce, compose, partition
    from operator import add, itemgetter

Anonymous functions, map and filter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    x = range(10)
    map(lambda x: x*x, filter(lambda x: x%2==0, x))




.. parsed-literal::

    [0, 4, 16, 36, 64]



Reduce, accumulate and fold
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    reduce(add, x, 0)




.. parsed-literal::

    45



Flatmap and function composition
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    flatmap = compose(concat, map)

.. code:: python

    from string import split
    
    s = ["hello world", "this is the end"]
    print list(map(split, s))
    print list(flatmap(split, s))


.. parsed-literal::

    [['hello', 'world'], ['this', 'is', 'the', 'end']]
    ['hello', 'world', 'this', 'is', 'the', 'end']


Working with key-value pairs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    s = 'aabaabcdeda'
    a = [(_, 1) for _ in s]
    print a


.. parsed-literal::

    [('a', 1), ('a', 1), ('b', 1), ('a', 1), ('a', 1), ('b', 1), ('c', 1), ('d', 1), ('e', 1), ('d', 1), ('a', 1)]


.. code:: python

    [item[0] for item in g.itervalues()]




.. parsed-literal::

    [('a', 1), ('c', 1), ('b', 1), ('e', 1), ('d', 1)]



.. code:: python

    groupby(itemgetter(0), a, )




.. parsed-literal::

    {'a': [('a', 1), ('a', 1), ('a', 1), ('a', 1), ('a', 1)],
     'b': [('b', 1), ('b', 1)],
     'c': [('c', 1)],
     'd': [('d', 1), ('d', 1)],
     'e': [('e', 1)]}



.. code:: python

    countby(itemgetter(0), a)




.. parsed-literal::

    {'a': 5, 'b': 2, 'c': 1, 'd': 2, 'e': 1}



The Hadoop MapReduce workflow
-----------------------------

A Hadoop job consists of the input file(s) on HDFS, :math:`m` map tasks
and :math:`n` reduce tasks, and the output is :math:`n` files. The
stages of one map-reduce iteration are:

-  mapper (written by programmer)
-  combiner (written by programmer)
-  sort and shuffle (done by Hdaoop framework)
-  reduce (written by programmer)

At each such iteration, there is input read in from HDFS and given to
the mapper, and output written out to HDFS by the reducer. Optimizing
the MapReduce pipeline often consists of minimizing the I/O tranfers.

Illustrating ideas behind MapReduce with a toy example of counting the number of each character in a string
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Input
^^^^^

.. code:: python

    s = 'aabaabcdeda'

Map to create a key-value pair
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: python

    xs = map(lambda x: [x, 1], s)
    xs




.. parsed-literal::

    [['a', 1],
     ['a', 1],
     ['b', 1],
     ['a', 1],
     ['a', 1],
     ['b', 1],
     ['c', 1],
     ['d', 1],
     ['e', 1],
     ['d', 1],
     ['a', 1]]



Sort and shuffle (aggregate and transfer data)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: python

    xs = sorted(xs)
    ys = []
    seen = []
    for x in xs:
        if x[0] not in seen:
            seen.append(x[0])
            ys.append([x[0], [x[1]]])
        else:
            ys[-1][1].append(x[1])
    ys




.. parsed-literal::

    [['a', [1, 1, 1, 1, 1]], ['b', [1, 1]], ['c', [1]], ['d', [1, 1]], ['e', [1]]]



Reduce
~~~~~~

.. code:: python

    for y in ys:
        print y[0], reduce(add, y[1])


.. parsed-literal::

    a 5
    b 2
    c 1
    d 2
    e 1


Using Hadoop MapReduce
----------------------

We want to count the number of times each word occurs in a set of books.
We will do this in Python.

Start up Hadoop
^^^^^^^^^^^^^^^

.. code:: bash

    start-dfs.sh
    start-yarn.sh

This will generate a lot of chatter

.. code:: bash

    15/04/06 12:42:13 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    Starting namenodes on [localhost]
    localhost: starting namenode, logging to /usr/local/Cellar/hadoop/2.6.0/libexec/logs/hadoop-cliburn-namenode-lister.dulci.duhs.duke.edu.out
    localhost: starting datanode, logging to /usr/local/Cellar/hadoop/2.6.0/libexec/logs/hadoop-cliburn-datanode-lister.dulci.duhs.duke.edu.out
    Starting secondary namenodes [0.0.0.0]
    0.0.0.0: starting secondarynamenode, logging to /usr/local/Cellar/hadoop/2.6.0/libexec/logs/hadoop-cliburn-secondarynamenode-lister.dulci.duhs.duke.edu.out
    15/04/06 12:42:30 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    starting yarn daemons
    starting resourcemanager, logging to /usr/local/Cellar/hadoop/2.6.0/libexec/logs/yarn-cliburn-resourcemanager-lister.dulci.duhs.duke.edu.out
    localhost: starting nodemanager, logging to /usr/local/Cellar/hadoop/2.6.0/libexec/logs/yarn-cliburn-nodemanager-lister.dulci.duhs.duke.edu.out

Make user direcotry (first time only)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: bash

    hdfs dfs -mkdir /user
    hdfs dfs -mkdir /user/cliburn

Transfer files to HDFS
^^^^^^^^^^^^^^^^^^^^^^

.. code:: bash

    hadoop dfs -copyFromLocal books /user/cliburn/books

Write a mapper function
^^^^^^^^^^^^^^^^^^^^^^^

.. code:: python

    %%file mapper.py
    #!/usr/bin/env python
    
    import sys
    
    def read_input(file):
        for line in file:
            yield line.split()
    
    def main(sep='\t'):
        data = read_input(sys.stdin)
        for words in data:
            for word in words:
                print '%s%s%d' % (word, sep, 1)
    
    if __name__ == '__main__':
        main()


.. parsed-literal::

    Writing mapper.py


Write a reducer function
^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: python

    %%file reducer.py
    #!/usr/bin/env python
    
    from itertools import groupby
    from operator import itemgetter
    import sys
    
    def read_mapper_output(file, sep):
        for line in file:
            yield line.rstrip().split(sep, 1)
    
    def main(sep='\t'):
        data = read_mapper_output(sys.stdin, sep=sep)
        for word, group in groupby(data, itemgetter(0)):
            total_count = sum(int(count) for word, count in group)
            print '%s%s%d' % (word, sep, total_count)
    
    if __name__ == '__main__':
        main()


.. parsed-literal::

    Overwriting reducer.py


Make programs executable
^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: python

    ! chmod +x maper.py
    ! chmod +x reducer.py

Feed to Hadoop streaming
^^^^^^^^^^^^^^^^^^^^^^^^

The native language for Hadoop is Java, but Hadoop stremaing allows
custom prograsm in other langauges to write the mapper, combiner and
reducer functions. For full set of options, see
http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/HadoopStreaming.html

.. code:: bash

    hadoop jar $HADOOP_HOME/libexec/share/hadoop/tools/lib/hadoop-*streaming*.jar \
    -file ./mapper.py    -mapper ./mapper.py \
    -file ./reducer.py   -reducer ./reducer.py \
    -input /user/cliburn/books/* -output /user/cliburn/books-output

View results
^^^^^^^^^^^^

.. code:: bash

    hdfs dfs -ls /user/cliburn/books-output
    hdfs dfs -cat /user/cliburn/books-output/part-00000

Stopping haddop
^^^^^^^^^^^^^^^

./sbin/stop-yarn.sh ./sbin/stop-dfs.sh

Using MrJob
~~~~~~~~~~~

The Python module `mrjob <http://mrjob.readthedocs.org/en/latest/>`__
removes a lot of the boilerplate and can also send jobs to Amazon's
implemtation of Hadoop known as Elastic Map Reduce (EMR).

.. code:: python

    ! pip install mrjob 


.. parsed-literal::

    Requirement already satisfied (use --upgrade to upgrade): mrjob in /Users/cliburn/anaconda/lib/python2.7/site-packages
    Requirement already satisfied (use --upgrade to upgrade): boto>=2.2.0 in /Users/cliburn/anaconda/lib/python2.7/site-packages (from mrjob)
    Requirement already satisfied (use --upgrade to upgrade): PyYAML in /Users/cliburn/anaconda/lib/python2.7/site-packages (from mrjob)
    Requirement already satisfied (use --upgrade to upgrade): simplejson>=2.0.9 in /Users/cliburn/anaconda/lib/python2.7/site-packages (from mrjob)


.. code:: python

    %%file word_count.py
    # From http://mrjob.readthedocs.org/en/latest/guides/quickstart.html#writing-your-first-job
    from mrjob.job import MRJob
    
    class MRWordFrequencyCount(MRJob):
    
        def mapper(self, _, line):
            yield "chars", len(line)
            yield "words", len(line.split())
            yield "lines", 1
    
        def reducer(self, key, values):
            yield key, sum(values)
    
    
    if __name__ == '__main__':
        MRWordFrequencyCount.run()


.. parsed-literal::

    Writing word_count.py


Running the job
^^^^^^^^^^^^^^^

As a single Python process for debugging

.. code:: bash

    python word_count.py books/*

To run on Hadoop cluster

.. code:: bash

    python word_count.py -r hadoop books/*

To run on Amazon EMR using files on S3

.. code:: bash

    python word_count.py -r emr s3://<path_to_books>

Java version
~~~~~~~~~~~~

For comparison, here is the first Java version from the `official
tutorial <http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html>`__:

.. code:: java

    import java.io.IOException;
    import java.util.StringTokenizer;

    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.Mapper;
    import org.apache.hadoop.mapreduce.Reducer;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

    public class WordCount {

      public static class TokenizerMapper
           extends Mapper<Object, Text, Text, IntWritable>{

        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public void map(Object key, Text value, Context context
                        ) throws IOException, InterruptedException {
          StringTokenizer itr = new StringTokenizer(value.toString());
          while (itr.hasMoreTokens()) {
            word.set(itr.nextToken());
            context.write(word, one);
          }
        }
      }

      public static class IntSumReducer
           extends Reducer<Text,IntWritable,Text,IntWritable> {
        private IntWritable result = new IntWritable();

        public void reduce(Text key, Iterable<IntWritable> values,
                           Context context
                           ) throws IOException, InterruptedException {
          int sum = 0;
          for (IntWritable val : values) {
            sum += val.get();
          }
          result.set(sum);
          context.write(key, result);
        }
      }

      public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        job.setJarByClass(WordCount.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setCombinerClass(IntSumReducer.class);
        job.setReducerClass(IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
      }
    }

Hadoop MapReduce Patterns
~~~~~~~~~~~~~~~~~~~~~~~~~

Most Hadoop work flows are organized as several rounds of map/reduce -
this is known as job chaining. Because I/O is so expensive, chain
folding where jobs are rearranged to minimize inputs/outputs and job
merging where unrelated jobs using the same dataset are run togtether
are common. In ``mrjob``, job chaining is performed via the `steps
abstraction <http://mrjob.readthedocs.org/en/latest/guides/writing-mrjobs.html>`__.

There are several common patterns that are repeatedly used in Hadoop
MapReduce programs:

-  Summarization (e.g. sum, mean, counting)
-  Filtering (e.g. subsampling, removing poor quality items, top 10
   lists)
-  Data organization (e.g. converting to hiearhical format, binning)
-  Joins

While it is certinly possible, it will take a lot of work to code, debug
and optimize any non-trivial program using just MapReduce construct, for
example regularized logistic regression on a large data set. Hence, we
will switch our focus to more modern tools such as ``Spark`` and
``Impala`` that provide higher level abstractions and often greater
efficiency.

Spark
-----

Spark provides a much richer set of programming constructs and libraries
that greatly simplify concurrent programming. In addition, because Spark
data can be persistent over a session (unliike MapReduce which
reads/writes data at each step in the job chain), it can be much faster
for iteratvie programs and also enables interactive concurrent
programming. See `official
documenttion <https://spark.apache.org/docs/latest/index.html>`__ for
details, including `setting up on
Amazon <https://spark.apache.org/docs/latest/ec2-scripts.html>`__. This
`article <http://aws.amazon.com/articles/Elastic-MapReduce/4926593393724923>`__
on how to set up Spark on EMR may also be helpful.

Very conceniently for learning, Spark provides an REPL shell where you
can interactively type and run Spark programs. For example, this will
open a Spark shell as an IPython Notebook (if spark is installed and
pyspark is on your path):

.. code:: bash

    IPYTHON_OPTS="notebook" pyspark

To whet your appetite, here is the stadnalone Spark version for the word
count program.

.. code:: python

    %%file spark_count.py
    
    from pyspark import SparkConf, SparkContext
    conf = SparkConf().setMaster("local").setAppName("Word Count")
    sc = SparkContext(conf = conf)
    
    rdd = sc.textFile("<path_to_books>")
    words = rdd.flatMap(lambda x: x.split())
    result = words.countByValue()


.. parsed-literal::

    Writing spark_count.py


And this is run by typing on the command line

.. code:: bash

    bin/spark-submit spark_count.py

Of course, ``spark-submit`` has many options that can be provided to
configure the job.

