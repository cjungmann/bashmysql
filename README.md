# BASH MySQL

This program processes a MySQL query to provide access to the columns'
metadata as well as the query's results.

The program serves two purposes.  Primarily, it processes a MySQL query
to give convenient access to the query results and the columns' metadata.

Second, it is an experiment with BASH programming.  I implement a few
ideas I'm working on:

- BASH variables in one function are visible to functions called by the
  function. I'm exploiting that characteristic by creating informational
  arrays and invoking a callback to announce the data availability.
- I'm using the `source` builtin BASH function to map the `bashmysql`
  functions directly into the calling scope.  That makes it possible to
  make the functions in `bashmysql` like a library for accessing the
  data.  Using `source` is recommended, but not required.
- The arrays are multi-dimensional, with a set of control characters
  to extract the different dimensions.
- I'm trying a bit more involved parameter processing, not needing them
  to come in a specific order (outside of the order of the database name
  and query text.

## Operation

The main program runs the query twice, once to collect columns' metadata
after having adding conditions to create an empty resultset, and again
to get the data rows of the query.  The two passes are not strictly
necessary, but the format of the output that includes the metadata is
much longer and more complicated to parse.

The metadata is organized into array **META_COLUMNS**.  The
*META_COLUMNS* information is used to build a reference array,
**RESULT_COLUMN_NAMES**, the contents of which are used to identify
the elements of the next array, **RESULT_ROWS**.

*RESULT_ROWS* is a multidimensional array, using the value of **RSEP**
(a single character, `\a` by default) to separate rows.  Each row is a
string with **FSEP** (`\006` by default) separating the fields.

## Usage

The program is designed to run within another script that provides a
callback function through which the results are accessed.

The common process is to provide a callback function that freezes the
scope while the calling script uses the data.  The examples in this
guide will name the primary callback function `data_ready` for lack
of a more accurately descriptive name.

### Sample1: Simplest Format

This first example uses the least efficient but easiest data access
function, `iterate_rows`.  `iterate_rows` takes another callback
function name, `row_user` in this example, which it will call once
for each result row.  Each invocation of `row_user` will be able to
read an associated array, **CURROW** to get the field information.

It's easier to use `iterate_rows` because the fields can be accessed
by name, but it's less efficient because each iteration must clear
and reset all the associated data.

The first example does not use the recommended *source* command. As a
result, the functions and variables in `bashmysql` are not imported
into the current shell.  The functions *row_user* and *data_ready*
must be exported to be visible to `bashmysql`.

~~~sh
#!/bin/bash

row_user()
{
   varname="${CURROW['variable']}"
   varval="${CURROW['value']}"

   echo "'${varname}'='${varval}'"
}

data_ready()
{
   iterate_rows row_user
}

export -f row_user
export -f data_ready

./bashmysql sys "SELECT * FROM sys_config" -c data_ready
~~~

### Sample2: Raw Data Access

When *data_ready* is called by `bashmysql` and in turn calls other
functions, *data_ready* and the called functions have direct access
to the data in the `bashmysql` function that called *data_ready*.

This program is quite a bit longer than sample1, so it is left to
the reader to access the file from the repository.

There are several variables that are available:

- **RESULT_COLUMN_NAMES** is an array of column names in the order
  of the SELECT clause of the query.
- **RESULT_ROWS** is the array of rows.  The fields in each row are
  separated by the character in *FSEP*.
- **META_COLUMNS** is an array of name:value elements that record the
  metadata MySQL returns about the result columns.  This data used
  to build the RESULT_COLUMN_NAMES and RESULT_ROWS arrays, but is
  not yet otherwise used.  The metadata would be useful for random
  queries for which the column characteristics are not known in
  advance.
- **RSEP**, **FSEP**, and **VSEP** for Record, Field, and Value
  SEParators.


### Why Not Export Functions?

If `bashmysql` is called without `source`, the `data_ready` and
`row_user` functions will be invisible.  Another solution that works
in this situation is to export the callback functions.  However,
more complicated programs may want save for later use the information
in a pass through the results.




The sample program, `get_table_info` demonstrates different methods
of using `bashmysql`.


