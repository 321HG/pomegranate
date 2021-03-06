.. _ooc:

Out of Core Learning
====================

Sometimes datasets which we'd like to train on can't fit in memory but we'd still like to get an exact update. pomegranate supports out of core training to allow this, by allowing models to summarize batches of data into sufficient statistics and then later on using these sufficient statistics to get an exact update for model parameters. These are done through the methods ```model.summarize``` and ```model.from_summaries```. Let's see an example of using it to update a normal distribution.

.. code-block:: python

	>>> from pomegranate import *
	>>> import numpy
	>>>
	>>> a = NormalDistribution(1, 1)
	>>> b = NormalDistribution(1, 1)
	>>> X = numpy.random.normal(3, 5, size=(5000,))
	>>> 
	>>> a.fit(X)
	>>> a
	{
	    "frozen" :false,
	    "class" :"Distribution",
	    "parameters" :[
	        3.012692830297519,
	        4.972082359070984
	    ],
	    "name" :"NormalDistribution"
	}
	>>> for i in range(5):
	>>>     b.summarize(X[i*1000:(i+1)*1000])
	>>> b.from_summaries()
	>>> b
	{
	    "frozen" :false,
	    "class" :"Distribution",
	    "parameters" :[
	        3.01269283029752,
	        4.972082359070983
	    ],
	    "name" :"NormalDistribution"
	}

This is a simple example with a simple distribution, but all models and model stacks support this type of learning. Lets next look at a simple Bayesian network.

.. code-block::python

	>>> from pomegranate import *
	>>> import numpy
	>>>
	>>> d1 = DiscreteDistribution({0: 0.25, 1: 0.75})
	>>> d2 = DiscreteDistribution({0: 0.45, 1: 0.55})
	>>> d3 = ConditionalProbabilityTable([[0, 0, 0, 0.02], 
								  [0, 0, 1, 0.98],
								  [0, 1, 0, 0.15],
								  [0, 1, 1, 0.85],
								  [1, 0, 0, 0.33],
								  [1, 0, 1, 0.67],
								  [1, 1, 0, 0.89],
								  [1, 1, 1, 0.11]], [d1, d2])
	>>>
	>>> d4 = ConditionalProbabilityTable([[0, 0, 0.4], 
                                  [0, 1, 0.6],
                                  [1, 0, 0.3],
                                  [1, 1, 0.7]], [d3]) 
    >>>
	>>> s1 = State(d1, name="s1")
	>>> s2 = State(d2, name="s2")
	>>> s3 = State(d3, name="s3")
	>>> s4 = State(d4, name="s4")
	>>>
	>>> model = BayesianNetwork()
	>>> model.add_nodes(s1, s2, s3, s4)
	>>> model.add_edge(s1, s3)
	>>> model.add_edge(s2, s3)
	>>> model.add_edge(s3, s4)
	>>> model.bake()
	>>> model2 = model.copy()
	>>>
	>>> X = numpy.random.randint(2, size=(10000, 4))
	>>> print model.states[0].distribution.equals( model2.states[0].distribution )
	True
	>>> model.fit(X)
	>>> print model.states[0].distribution.equals( model2.states[0].distribution )
	False
	>>> model2.summarize(X[:2500])
	>>> model2.summarize(X[2500:5000])
	>>> model2.summarize(X[5000:7500])
	>>> model2.summarize(X[7500:])
	>>> model2.from_summaries()
	>>>
	>>> print model.states[0].distribution.equals( model2.states[0].distribution )
	True

We can see that before fitting to any data, the distribution in one of the states is equal for both. After fitting the first distribution they become different as would be expected. After fitting the second one through summarize the distributions become equal again, showing that it is recovering an exact update.

It's easy to see how one could use this to update models which don't use Expectation Maximization (EM) to train, since it is an iterative algorithm. For algorithms which use EM to train there is a ```fit``` wrapper which will allow you to load up batches of data from a numpy memory map to train on automatically. 

FAQ
---

Q. What data storage types are able to be used with out of core training?

A. Currently only stored numpy arrays (.npy files) that can be read as memory maps using `numpy.load('data.npy', mmap_mode='r')` are supported for data that truly can't be loaded into memory.


Q. Does out of core learning give exact or approximate updates?

A. It gives exact updates. Sufficient statistics are collected for each of the batches and are equal to the sufficient statistics that one would get from the full dataset. 
