---
layout: default
description: ArangoDB's graph traversals are executed on the server
---
HTTP Interface for Traversals
=============================

### Traversals

ArangoDB's graph traversals are executed on the server. Traversals can be 
initiated by clients by sending the traversal description for execution to
the server.

Traversals in ArangoDB are used to walk over a graph
stored in one [edge collection](../appendix-glossary.html#edge-collection). It can easily be described
which edges of the graph should be followed and which actions
should be performed on each visited vertex.
Furthermore the ordering of visiting the nodes can be
specified, for instance depth-first or breadth-first search
are offered.

Executing Traversals via HTTP
-----------------------------

{% docublock JSF_HTTP_API_TRAVERSAL %}

All examples were using this graph:

![Persons relation Example Graph](../images/knows_graph.png)
