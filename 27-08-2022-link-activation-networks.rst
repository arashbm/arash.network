.. post:: Aug 25, 2022
   :tags: reticula
   :author: Arash Badie-Modiri

Random link-activation temporal network with Reticula
=====================================================

For the past few years, I have been studying properties of temporal networks and
how certain inhomogeneities affect the nature and extent of connectivity in
them. The most straightforward avenue of attacking this problem is to construct
a random temporal network using a generative model that includes the desired
spatial or temporal property, for example burstiness or degree inhomogeneity,
and comparing the connectivity to temporal networks generated through methods
that don't include that property. In this post I'll go through my current
go-to family of models, link-activation temporal networks, as well as how you
can use them in the `Reticula`_ network analysis library.

.. _Reticula: https://reticula.network/


In short, link-activation models work in two steps: First, you generate a static
network representing the spatial projection, the time aggregate, of the final
temporal network. This indicates in a general way who is connected to whom. This
static network can also be the result of a previous study or an observation of a
real-world phenomenon.

In the next step, you generate activation times for each of the links in the
static network based on some process. This can be as simple as a Poisson process
(with exponential inter-event times) which assumes events happen independently
at random at a constant rate, or maybe some other renewal process with a
different inter-event time distribution such as a power-law with minimum cutoff,
or a non-Markovian process such as Hawkes univariate self-exciting process, as
long as activations of different links are independent events.

Let's see a quick example:

.. code-block:: python
  :caption: Simple random link-activation network

  import reticula as ret

  state = ret.mersenne_twister(seed=42)

  # generate a static random G(n, p) network
  static = ret.random_gnp_graph[ret.int64](n=100, p=0.05, random_state=state)

  # define an inter-event time distribution
  mean_iet = 1.0
  iet_dist = ret.exponential_distribution[ret.double](1/mean_iet)

  # generate a random link-activation
  temp = ret.random_link_activation_temporal_network(static, max_t=1024,
    iet_dist=iet_dist, res_dist=iet_dist, random_state=state)

The function :py:`mersenne_twister()` creates a pseudo-random number generator
that can be used by various functions in Reticula that need a source of
randomness. After that we generate a random :math:`G(n, p)` graph with 64-bit
signed integer vertices, as indicated by the type parameter :py:`ret.int64`.

In the next step, we define our desired inter-event time distribution. Here, we
use an exponential distribution that generated double-precision floating-points,
indicated by the type parameter :py:`ret.double`. The rate parameter of the
exponential distribution is by definition the reciprocal of the mean
inter-event time.

The last line is, however, where the magic happens. In addition to the
inter-event time distribution :py:`iet_dist`, the function
:py:`random_link_activation_temporal_network()` needs a `residual event-time
distribution <https://en.wikipedia.org/wiki/Residual_time>`_, which is used to
draw the time of the first events. The idea here is to generate a random network
that is indistinguishable from an observation of a temporal network with the
given inter-event time distribution starting at a random starting time. The
residual event-time distribution is the distribution of time to the first
activation of the activation process from a random point in time.

For some activation processes, such as :py:`power_law_with_specified_mean`,
Reticula has pre-defined residual distribution in form of
:py:`residual_power_law_with_specified_mean`. For the special case of the
exponential distribution, however, the residual event time distribution is
`identical to the inter-event time distribution
<https://en.wikipedia.org/wiki/Exponential_distribution#Memorylessness>`_. We
can just re-use the variable :py:`iet_dist` from before.

The example above can be easily modified to suit your needs. Let's say you are
interested in studying self-exciting processes. The previous example can be
updated as follows:

.. code-block:: python
  :caption: Random link-activation network with Hawkes self-exciting process

  import reticula as ret

  state = ret.mersenne_twister(seed=42)

  # generate a static random G(n, p) network
  static = ret.random_gnp_graph[ret.int64](n=100, p=0.05, random_state=state)

  # define an inter-event time distribution
  iet_dist = ret.hawkes_univariate_exponential[ret.double](
    mu=0.2, alpha=0.8, theta=0.5)

  # generate a random link-activation
  temp = ret.random_link_activation_temporal_network(static, max_t=1024,
            iet_dist=iet_dist, random_state=state)

This generates a temporal network based on a random :math:`G(n, p)` graph and
inter-event times driven from a Hawkes univariate exponential self-exciting
process. The parameters :py:`mu`, :py:`alpha` and :py:`theta` indicate
background intensity (rate) of events, infectivity factor and rate parameter of
the delay respectively.

Similarly you can change the static network in the variable :py:`static` to
something else, e.g. a random :math:`k`-regular network, a Barabási–Albert
network, or a Random expected degree-sequence network with an arbitrary
degree-sequence of your choice. This static base network can be any directed or
undirected dyadic networks or hypergraph.

You can see an example of this method being used in the paper `Directed
percolation in random temporal network models with heterogeneities
<https://journals.aps.org/pre/abstract/10.1103/PhysRevE.105.054313>`_, where the
we compared the critial threshold of reachability between different temporal
networks generated using different static models and temporal activation
proceses.
