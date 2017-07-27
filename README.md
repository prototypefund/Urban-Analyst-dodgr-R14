<!-- README.md is generated from README.Rmd. Please edit that file -->
[![Build Status](https://travis-ci.org/mpadge/pqspr.svg)](https://travis-ci.org/mpadge/pqspr) [![Project Status: Concept - Minimal or no implementation has been done yet.](http://www.repostatus.org/badges/0.1.0/concept.svg)](http://www.repostatus.org/#concept) [![CRAN\_Status\_Badge](http://www.r-pkg.org/badges/version/pqspr)](http://cran.r-project.org/web/packages/pqspr)

didg
====

Calculated pairwise distances on dual-weighted directed graphs using Priority Queue Shortest Paths. Dual-weighted graphs are ones where shortest paths are evaluated according to one weight, while distances along paths are calculating using a different weight. A canonical example is routing along a street network according to some kind of priority weighting for a particular mode of transport. The weights determine the route, while the desired output is a simple distance.

The shortest path algorithm relies on priority queue code adapted from original code by Shane Saunders available here <http://www.cosc.canterbury.ac.nz/tad.takaoka/alg/spalgs/spalgs.html>

Installation
------------

You can install `didg` from github with:

``` r
# install.packages("devtools")
devtools::install_github("mpadge/pqspr")
```

Usage
-----

A graph or network in `didr` is represented as a flat table (`data.frame`, `tibble`, `data.table`, whatever) of minimally four columns: `from`, `to`, `weight`, and `distance`. The first two can be of arbitrary form (`numeric` or `character`); `weight` is used to evaluate the shortest paths, and the desired distances are evaluated by summing the values of `distance` along those paths. For a street network example, `weight` will generally be the actual distance multiplied by a priority weighting for a given mode of transport and type of way, while `distance` will be the pysical distance.

The primary function of `didr` is `didr_dists()`, which accepts the three main arguments of

1.  `graph` - the flat table described above;
2.  `from` - a list of origin points for which distances are to be calcualted; and
3.  `to` - a list of equivalent destination points. If missing, pairwise distances are calculated between all points in `from`.

For spatial graphs in which `graph` has additional longitude and latitude coordinates, `from` and `to` may also be given as lists of coordinates; otherwise they must match onto names or values given in `graph$from` and `graph$to`. Coordinates in `from` and `to` that do not exactly match on to values given in `graph` will be mapped on to the nearest `graph` nodes.

Timing Comparison
-----------------

Create a street network using the github repo [`osmprob`](https://github.com/osm-router/osmprob):

``` r
devtools::install_github ("osm-router/osmprob")
```

The function `download_graph()` gets the street network within the bounding box defined by start and end points.

``` r
start_pt <- c (-74.00150, 40.74178)
end_pt <- c (-74.07889, 40.71113)
graph <- download_graph (start_pt, end_pt)
```

The result has a `$original` graph containing all vertices and a `$compact` graph with redundant vertices removed. We'll perform routing on the latter which has this many edges:

``` r
nrow (graph$compact)
```

Both of these graphs are simple data frames detailing all edges. Some light wrangling is now necessary to prepare both the `igraph` and the simple structure submitted to the `pqspr` routine:

``` r
edges <- cbind (graph$compact$from_id, graph$compact$to_id)
nodes <- unique (as.vector (edges)) # used below in test comparison
edges <- as.vector (t (edges))
igr <- igraph::make_directed_graph (edges)
igraph::E (igr)$weight <- graph$compact$d_weighted

graph <- graph$compact
indx <- which (names (graph) %in% c ("from_id", "to_id", "d", "d_weighted"))
graph <- graph [, indx]
graph$from_id <- paste0 (graph$from_id)
graph$to_id <- paste0 (graph$to_id)
```

Then the final timing comparison between `igraph::distances()`, which returns a matrix of distances between all vertices, and the equivalent and only function of this package, `test()`:

``` r
rbenchmark::benchmark (
                       d <- test (graph, heap = "FHeap"),
                       d <- test (graph, heap = "BHeap"),
                       d <- test (graph, heap = "TriHeap"),
                       d <- test (graph, heap = "TriHeapExt"),
                       d <- test (graph, heap = "Heap23"),
                   igraph::distances (igr, v = nodes, to = nodes, mode = "out"),
                       replications = 10)
```

And these priority queue routines are over ten times faster than the `igraph` equivalent. The default priority queue is a Fibonacci heap (`FHeap`), with alternative options of `Radix`, `Tri`, and `Heap23`.
