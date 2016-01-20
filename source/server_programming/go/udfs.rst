User-Defined Functions
======================

This section describes how to write a UDF in Go. It first shows the basic
interface of defining UDFs, and then describes utilities among it.

Overview
--------

A struct satisfying the following interface can be a UDF::

    type UDF interface {
        // Call calls the UDF.
        Call(*core.Context, ...data.Value) (data.Value, error)

        // Accept checks if the function accepts the given number of arguments
        // excluding core.Context.
        Accept(arity int) bool

        // IsAggregationParameter returns true if the k-th parameter expects
        // aggregated values. A UDF with Accept(n) == true is an aggregate
        // function if and only if this function returns true for one or more
        // values of k in the range 0, ..., n-1.
        IsAggregationParameter(k int) bool
    }

``Call`` method receives ``*core.Context`` and multiple ``data.Value`` as its
arguments. ``*core.Context`` contains the information of the current processing
context. ``data.Value`` represents a value used in BQL and can be any of
:ref:`built-in types <bql_types_types>`.

``Accept`` method verifies if the UDF accept the specific number of arguments.
It can return true for multiple arities as long as it can receive the given
number of arguments.

``IsAggregationParameter`` checks if k-th argument is an aggregation parameter.

A UDF can be registered by ``RegisterGlobalUDF`` or ``MustRegisterGlobalUDF``
function defined in ``gopkg.in/sensorbee/sensorbee.v0/bql/udf`` package.
``MustRegisterGlobalUDF`` is same as ``RegisterGlobalUDF`` but panics on failure
instead of returning an error. These functions are usually called from ``init``
function::

    type MyUDF struct {
        ...
    }

    ... implementation of required methods ...

    func init() {
        udf.MustRegisterGlobalUDF("my_udf", &MyUDF{})
    }

As it can be inferred from this example, a UDF itself should be stateless since
it only registers one instance of a struct as a UDF and it'll globally be shared.
Stateful data processing can be achieved by the combination of UDFs and UDSs,
which is described in the following sections.

The registered UDF is looked up based on its name and the number of argument
passed to it.

::

    SELECT RSTREAM my_udf(arg1, arg2) FROM stream [RANGE 1 TUPLES];

In this ``SELECT``, a UDF having the name ``my_udf`` is looked up first. After
that, its ``Accept``method  is called with 2 and ``my_udf`` is actually selected
if ``Accept(2)`` returned true. ``IsAggregationParameter`` method is
additionally called on each argument to see if the argument needs to be an
aggregation parameter. Then, if there's no mismatch, ``my_udf`` is finally
called.

.. note::

    A UDF doesn't have a schema at the moment, so any error regarding types of
    arguments will not be reported until the statement calling the UDF actually
    processes a tuple.

Generic UDFs
------------

SensorBee provides a helper function to register a regular Go function as a UDF
without implementing ``UDF`` interface explicitly.

::

    func Inc(v int) int {
        return v + 1
    }

This function ``Inc`` can be transformed into a UDF by ``ConvertGeneric``
or ``MustConvertGeneric`` function defined in
``gopkg.in/sensorbee/sensorbee.v0/bql/udf`` package. By combining it with
``RegisterGlobalUDF``, ``Inc`` function can be registered as a UDF::

    func init() {
        udf.MustRegisterGlobalUDF("inc", udf.MustConvertGeneric(Inc))
    }

Although this approach is handy, there could be some overhead compared to a UDF
implemented in the regular way. Most of such overhead comes from type checking
and conversions.

Functions passed to ``ConvertGeneric`` need to satisfy some restrictions on
the form of their argument and return value types. Each restriction is described
in the following subsections.

Form of Arguments
^^^^^^^^^^^^^^^^^

In terms of valid argument forms, there're some rules to follow:

#. A Go function can receive ``*core.Context`` as the first argument, or can omit it.
#. A function can have any number of arguments including 0 argument as long as Go accepts them.
#. A function can be variadic with or without non-variadic parameters.

There're basically eight (four times two, whether a function has
``*core.Context`` or not) forms of arguments (return values are
intentionally omitted for clarity):

* Functions receiving no argument in BQL (e.g. ``my_udf()``)

    1. ``func(*core.Context)``: A function only receiving ``*core.Context``
    2. ``func()``: A function having no argument and not receiving ``*core.Context``, either

* Functions having non-variadic arguments but no variadic arguments

    3. ``func(*core.Context, T1, T2, ..., Tn)``
    4. ``func(T1, T2, ..., Tn)``

* Functions having variadic arguments but no non-variadic arguments

    5. ``func(*core.Context, ...T)``
    6. ``func(...T)``

* Functions having both variadic and non-variadic arguments

    7. ``func(*core.Context, T1, T2, ..., Tn, ...Tn+1)``
    8. ``func(T1, T2, ..., Tn, ...Tn+1)``

Followings are examples of invalid function signatures:

* ``func(T, *core.Context)``: ``*core.Context`` must be the first argument.
* ``func(NonSupportedType)``: Only supported types, which will be explained later, can be used.

Although return values are omitted from all the examples above, they're actually
required. The next subsection explains how to define valid return values.

Form of Return Values
^^^^^^^^^^^^^^^^^^^^^

All functions need to have return values. There're two forms of return values:

* ``func(...) R``
* ``func(...) (R, error)``

All other forms are invalid:

* ``func(...)``
* ``func(...) error``
* ``func(...) NonSupportedType``

Valid types of return values are same as the valid types of arguments, and
they'll be listed in the following subsection.

Valid Value Types
^^^^^^^^^^^^^^^^^

The list of Go types that can be used for arguments and the return value is as
follows:

* ``bool``
* signed integers: ``int``, ``int8``, ``int16``, ``int32``, ``int64``
* unsigned integers: ``uint``, ``uint8``, ``uint16``, ``uint32``, ``uint64``
* ``float32``, ``float64``
* ``string``
* ``time.Time``
* data: ``data.Bool``, ``data.Int``, ``data.Float``, ``data.String``,
  ``data.Blob``, ``data.Timestamp``, ``data.Array``, ``data.Map``, ``data.Value``
* A slice of any type above, including ``data.Value``

``data.Value`` can be used as a semi-variant type, which will receive all types
above.

When the argument type and the actual value type are different, weak type
conversion are applied to values. Conversions are basically done by
``data.ToXXX`` functions (see godoc comments of each function in
data/type_conversions.go). For example, ``func inc(i int) int`` can be called by
``inc('3')`` in a BQL statement and it'll return 4. If a strict type checking
or custom type conversion is required, receive values as ``data.Value`` and
manually check or convert types, or define the UDF in the regular way.

Examples of Valid Go Functions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Following functions can be converted to UDFs by ``ConvertGeneric`` or
``MustConvertGeneric`` function:

* ``func rand() int``
* ``func pow(*core.Context, float32, float32) (float32, error)``
* ``func join(*core.Context, ...string) string``
* ``func format(string, ...data.Value) (string, error)``
* ``func keys(data.Map) []string``

Developing a UDF
----------------

The basic development flow of a UDF is as follows:

#. Create a git repository for a UDF
#. Implement the UDF
#. Create a plugin subpackage in the repository

Create a Git Repository for a UDF
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The UDF is written in Go, so it needs to be a valid git repository (or a
repository of other version control systems). One repository may provide
multiple UDFs. However, since Go is very well designed to provide packages in
a fine-grained manner, each repository should only provide a minimum set of
UDFs that are logically related and make sense to be in the same repository.

Implement the UDF
^^^^^^^^^^^^^^^^^

The next step is to implement the UDF. There's no restiction on which packages
to use.

Functions or structs that are registered to the SensorBee server needs to be
referred by the plugin subpackage, which is described in the next subsection.
Thus, names of those symbols need to start with a capital letter.

In this step, the UDF shouldn't be registered to the SensorBee server yet.

Create a Plugin Subpackage in the Repository
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It is highly recommended that the repository have a separate package which only
registers UDFs to the SensorBee server. There's usually one file named "plugin.go"
and it only contains a series of ``RegisterGlobalUDF`` calls in ``init``
function. For instance, if the repository only provides one UDF, the contents of
"plugin.go" would be something like::

    // in github.com/user/myudf/plugin/plugin.go
    package plugin

    import (
        "gopkg.in/sensorbee/sensorbee.v0/bql/udf"
        "github.com/user/myudf"
    )

    func init() {
        udf.MustRegisterGlobalUDF("my_udf", &myudf.MyUDF{})
    }

There're two reasons to have a plugin subpackage separated from the
implementation of UDFs. Firstly, by separating them, other Go packages can
import the UDFs implementation to use the package as a library without
registering them to SensorBee. Secondly, having a separated plugin package
allows a user to register a UDF with a different name. This is especially useful
when names of UDFs conflict each other.

To use the example plugin above, "github.com/user/myudf/plugin" needs to be
added to the plugin path list of SensorBee.

Repository Organization
^^^^^^^^^^^^^^^^^^^^^^^

The typical organization of the repository is

* github.com/user/repo

    * README: description and the usage of the UDF
    * .go files: implementation of the UDF
    * plugin/: a subpackage for the plugin registration

        * plugin.go

    * othersubpackages/: there can be optional subpackages

An Example
----------

TODO

Dynamic Loading
---------------

Dynamic loading of UDFs written in Go isn't supported at the moment because
Go doesn't officially support loading packages dynamically.