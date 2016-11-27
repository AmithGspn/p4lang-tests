# Introduction

This repository includes sample P4 programs, publicly available, plus
commands used to analyze them, and the output files those commands
produce.

The section "Output files of `p4-graphs`" below contains some
documentation to aid in interpreting the output files of that program.


## Related repositories

* https://github.com/p4lang/p4-hlir - Python source code for
      `p4-validate` and `p4-graphs` programs that parse P4 source code
      and produce HLIR (High Level Intermediate Representation) of the
      program.  This is a collection of Python data structures
      representing things in the program like headers, tables,
      actions, conditional expressions, etc. plus dependencies and
      relationships between them.

* https://github.com/p4lang/switch - source for switch.p4, copied into
      this repository in directory
      [v1.0.3/switch-orig/](v1.0.3/switch-orig/)


## Output files of `p4-graphs`

This command:

    p4-graphs <basename>.p4

produces several output files, if it successfully parses the input
file, and the p4 program passes several semantic checks:

* Parse graph in files: <basename>.parser.{dot,png}
* Table control flow graph in files: <basename>.tables.{dot,png}
* Table dependency graphs in files:
  <basename>.ingress.tables_dep.{dot,png} for ingress pipeline, and
  <basename>.egress.tables_dep.{dot,png} for egress pipeline.

The `.dot` files are input files in the syntax expected by the `dot`
program in the GraphViz package.  They are used to generate the
corresponding `.png` files.

More details about each of these graphs is given below.


### Parse graph

Files: <basename>.parser.{dot,png}

Command line option: --parser

Primary functions in `p4-graphs` that generates them:
`export_parse_graph` and `dump_parser` in file source file dot.py.

Small example for mtag-edge.p4:
[v1.0.3/mtag/out1-unchanged/mtag-edge.parser.png](v1.0.3/mtag/out1-unchanged/mtag-edge.parser.png)

Larger example for switch.p4:
[v1.0.3/switch-orig/out1-unchanged/switch.parser.png](v1.0.3/switch-orig/out1-unchanged/switch.parser.png)

Parse graphs give good detailed representations of the parser nodes
and the field values used to decide which parse node to go to next.
They contain nothing about tables or actions.  Each parse graph begins
with the packet arriving from the wire, and ends with the first table
access.

TBD: Give 'legend' explaining shapes, styles, and colors of nodes and
edges.

TBD: Link to example, calling out pieces of it and what they mean.


### Table control flow graph

Files: <basename>.tables.{dot,png}

Command line option: --table

Primary functions in `p4-graphs` that generates them:
`export_table_graph` and `dump_table`

Small example for mtag-edge.p4:
[v1.0.3/mtag/out1-unchanged/mtag-edge.tables.png](v1.0.3/mtag/out1-unchanged/mtag-edge.tables.png)

Larger example for switch.p4:
[v1.0.3/switch-orig/out1-unchanged/switch.tables.png](v1.0.3/switch-orig/out1-unchanged/switch.tables.png)

TBD: Brief explanation of graph contents.

TBD: Give 'legend' explaining shapes, styles, and colors of nodes and
edges.

TBD: Link to example, calling out pieces of it and what they mean.

Table control flow graphs have these elements in them, from looking at
the output of p4-v1.0.3/mtag-edge.p4 and comparing it to the source
code.

Legend:

* double-circled node - Used for special nodes 'buffer', representing
    buffering point at end of ingress pipeline, just before egress
    pipeline, and for node 'egress' at end of egress pipeline.  My
    guess is that it is probably not used for any other nodes in this
    graph.

* ellipse node - Used for tables.  Each has one solid arrowed line
    leading out of it, labeled by the name of one action of the table.

* rectangle node - Used for conditional expressions in an 'if'
    statement.  The box contains the conditional expression.  Leading
    out of the node are two solid lines.  The one ending with a circle
    filled in black is the 'true' / 'then' branch.  The one ending
    with a circle filled in white is the 'false' / 'else' branch.

Nodes with ellipses around them include the following, and correspond
1 for 1 with 'table' definition in the .p4 source code, with the same
names:

    strip_mtag
    identify_port
    local_switching
    mTag_table
    egress_check
    egress_meter
    meter_policy

Each of those nodes has one or more lines with arrowheads leading
downwards out of them.  Each is labeled with the name of one of the
actions defined for the table.  Often the destinations of these arrows
are the same next node, but they can differ.  The one example I have
seen of them differing so far is for the table 'egress_meter', where
the arrowed line labeled 'hit' goes to the table 'meter_policy', but
the arrowed line labeled 'miss' skips that table and goes to the
'egress' node.

NOTE: In the standard output of the p4-graphs command for input file
mtag-edge.p4, it shows the following:

    parsing successful
    semantic checking successful
    Header type standard_metadata_t not byte-aligned, adding padding
    Generating files in directory /home/andy/p4/andy-tests/mtag-v1.0.2
    
    TABLE DEPENDENCIES...
    
    INGRESS PIPELINE
    ['strip_mtag', 'identify_port']
    ['local_switching']
    ['mTag_table']
    pipeline ingress requires at least 3 stages
    
    EGRESS PIPELINE
    ['egress_check', 'egress_meter']
    ['meter_policy']
    pipeline egress requires at least 2 stages

I am not sure yet how to interpret the output after the 'INGRESS
PIPELINE' line.

It could be that it is indicating that tables 'strip_mtag' and
'identify_port' can be searched concurrently, because even though the
'apply' operation for table 'identify_port' is sequentially after the
one for 'strip_mtag' in the p4 source code, the search key for
'identify_port' does not depend upon the actions of table
'strip_mtag'.

The search key for table identify_port contains only
standard_metadata.ingress_port.  That field is used several times in
the source code, but only to read it, never to modify it.

As an experiment, I added a field to table identify_port's search key
that is modified by an action of table strip_mtag, field
local_metadat.mtagged.  The standard output changed to the below:

    parsing successful
    semantic checking successful
    Header type standard_metadata_t not byte-aligned, adding padding
    Generating files in directory /home/andy/p4/andy-tests/mtag-v1.0.2
    
    TABLE DEPENDENCIES...
    
    INGRESS PIPELINE
    ['strip_mtag']
    ['identify_port']
    ['local_switching']
    ['mTag_table']
    pipeline ingress requires at least 4 stages
    
    EGRESS PIPELINE
    ['egress_check', 'egress_meter']
    ['meter_policy']
    pipeline egress requires at least 2 stages

That is a pretty strong indication that my guess above is correct
about dependencies.

local_switching search key does depend upon action of identify_port,
because action 'common_set_port_type' of table 'identify_port'
modifies field 'local_metadata.ingress_error', which is used in
'control ingress' in an 'if' condition to decide whether to do any
further table searches.

That field cannot be modified by actions of table 'strip_mtag', but
'strip_mtag' is sequentially before the if condition, and its actions
must be done before finishing ingress pipeline, so may as well do it
before the if condition is evaluated and then/else branches selected.

There is an 'if' checking value of field
'standard_metadata.egress_spec' after table local_switching is
applied, and that field can be modified by one of that table's
actions, so mTag_table access is dependent upon result of table
local_switching.

Table 'egress_meter' is applied in the egress pipeline sequentially
after table 'egress_check', but the search key of 'egress_meter'
cannot be modified by the actions of 'egress_check' (I verified this
by reading through the code).  Thus it makes sense that they would
show up in standard output as possibly concurrent.

Table 'meter_policy' access depends upon hit/miss result of table
'egress_meter', so is dependent upon it.  From examining the graph
<basename>.egress.tables_dep.png, it shows that the key of table
meter_policy includes field local_metadata.color, which is modified by
an action of table egress_meter.


### Table dependency graph

Files: <basename>.ingress.tables_dep.{dot,png} for ingress pipeline,
and <basename>.egress.tables_dep.{dot,png} for egress pipeline.  Also
a table of schedule accesses obeying the dependencies in the p4
program, and requiring the fewest number of stages possible, is given
in the standard output.

Command line option: --deps

Primary functions in `p4-graphs` that generates them:
`export_table_dependency_graph` in file dot.py, and
`build_table_graph_ingress`, `build_table_graph_egress`, and
`generate_graph` in file dependency_graph.py.

Small examples for mtag-edge.p4:
[v1.0.3/mtag/out1-unchanged/mtag-edge.ingress.tables_dep.png](v1.0.3/mtag/out1-unchanged/mtag-edge.ingress.tables_dep.png)
[v1.0.3/mtag/out1-unchanged/mtag-edge.egress.tables_dep.png](v1.0.3/mtag/out1-unchanged/mtag-edge.egress.tables_dep.png)

Larger examples for switch.p4:
[v1.0.3/switch-orig/out1-unchanged/switch.ingress.tables_dep.png](v1.0.3/switch-orig/out1-unchanged/switch.ingress.tables_dep.png)
[v1.0.3/switch-orig/out1-unchanged/switch.egress.tables_dep.png](v1.0.3/switch-orig/out1-unchanged/switch.egress.tables_dep.png)

TBD: Brief explanation of graph contents.

TBD: Give 'legend' explaining shapes, styles, and colors of nodes and
edges.

TBD: Link to example, calling out pieces of it and what they mean.

This file is focused on table dependencies of only the ingress
pipeline.

I think that when table B depends upon table A's actions, it
explicitly shows an arrowed line from A to B labeled with one or more
fields indicating the reason for the dependency, but I'll need to
double check that.

For the original mtag-edge.p4, it included a dotted line from
strip_mtag to identify_port, with no names on the line.  I think that
might be because they are sequential in the source code in that order,
but there are no dependencies between them.

I can verify that when I added the field local_metadata.was_mtagged,
modified by an action of table strip_mtag, to table identify_port's
search key, and re-ran p4-graphs, this file changed to have a solid
arrowed line from strip_mtag to identify_port, labeled with the name
of that field.  Great!

It also removed dependency lines out of strip_mtag that were there
before showing dependencies, I think because it is showing the fewest
dependencies needed to explain them.  That is, it seems to be a
transitive reduction of all dependencies.  Also nice!  It makes the
graph much less busy, but it also means that you cannot discover what
the other dependencies are from that graph, unless you modify the code
to eliminate some existing dependencies and re-run.  Still, I imagine
_not_ doing the transitive reduction would make these graphs difficult
to read, with a multitude of lines you do not care about most of the
time.