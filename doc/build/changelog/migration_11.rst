==============================
What's New in SQLAlchemy 1.1?
==============================

.. admonition:: About this Document

    This document describes changes between SQLAlchemy version 1.0,
    at the moment the current release series of SQLAlchemy,
    and SQLAlchemy version 1.1, which is the current development
    series of SQLAlchemy.

    As the 1.1 series is under development, issues that are targeted
    at this series can be seen under the
    `1.1 milestone <https://bitbucket.org/zzzeek/sqlalchemy/issues?milestone=1.1>`_.
    Please note that the set of issues within the milestone is not fixed;
    some issues may be moved to later milestones in order to allow
    for a timely release.

    Document last updated: September 2, 2015

Introduction
============

This guide introduces what's new in SQLAlchemy version 1.1,
and also documents changes which affect users migrating
their applications from the 1.0 series of SQLAlchemy to 1.1.

Please carefully review the sections on behavioral changes for
potentially backwards-incompatible changes in behavior.

Platform / Installer Changes
============================

Setuptools is now required for install
--------------------------------------

SQLAlchemy's ``setup.py`` file has for many years supported operation
both with Setuptools installed and without; supporting a "fallback" mode
that uses straight Distutils.  As a Setuptools-less Python environment is
now unheard of, and in order to support the featureset of Setuptools
more fully, in particular to support py.test's integration with it,
``setup.py`` now depends on Setuptools fully.

.. seealso::

	:ref:`installation`

:ticket:`3489`

Enabling / Disabling C Extension builds is only via environment variable
------------------------------------------------------------------------

The C Extensions build by default during install as long as it is possible.
To disable C extension builds, the ``DISABLE_SQLALCHEMY_CEXT`` environment
variable was made available as of SQLAlchemy 0.8.6 / 0.9.4.  The previous
approach of using the ``--without-cextensions`` argument has been removed,
as it relies on deprecated features of setuptools.

.. seealso::

	:ref:`c_extensions`

:ticket:`3500`


New Features and Improvements - ORM
===================================

.. _change_2677:

New Session lifecycle events
----------------------------

The :class:`.Session` has long supported events that allow some degree
of tracking of state changes to objects, including
:meth:`.SessionEvents.before_attach`, :meth:`.SessionEvents.after_attach`,
and :meth:`.SessionEvents.before_flush`.  The Session documentation also
documents major object states at :ref:`session_object_states`.  However,
there has never been system of tracking objects specifically as they
pass through these transitions.  Additionally, the status of "deleted" objects
has historically been murky as the objects act somewhere between
the "persistent" and "detached" states.

To clean up this area and allow the realm of session state transition
to be fully transparent, a new series of events have been added that
are intended to cover every possible way that an object might transition
between states, and additionally the "deleted" status has been given
its own official state name within the realm of session object states.

New State Transition Events
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Transitions between all states of an object such as :term:`persistent`,
:term:`pending` and others can now be intercepted in terms of a
session-level event intended to cover a specific transition.
Transitions as objects move into a :class:`.Session`, move out of a
:class:`.Session`, and even all the transitions which occur when the
transaction is rolled back using :meth:`.Session.rollback`
are explicitly present in the interface of :class:`.SessionEvents`.

In total, there are **ten new events**.  A summary of these events is in a
newly written documentation section :ref:`session_lifecycle_events`.


New Object State "deleted" is added, deleted objects no longer "persistent"
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :term:`persistent` state of an object in the :class:`.Session` has
always been documented as an object that has a valid database identity;
however in the case of objects that were deleted within a flush, they
have always been in a grey area where they are not really "detached"
from the :class:`.Session` yet, because they can still be restored
within a rollback, but are not really "persistent" because their database
identity has been deleted and they aren't present in the identity map.

To resolve this grey area given the new events, a new object state
:term:`deleted` is introduced.  This state exists between the "persistent" and
"detached" states.  An object that is marked for deletion via
:meth:`.Session.delete` remains in the "persistent" state until a flush
proceeds; at that point, it is removed from the identity map, moves
to the "deleted" state, and the :meth:`.SessionEvents.persistent_to_deleted`
hook is invoked.  If the :class:`.Session` object's transaction is rolled
back, the object is restored as persistent; the
:meth:`.SessionEvents.deleted_to_persistent` transition is called.  Otherwise
if the :class:`.Session` object's transaction is committed,
the :meth:`.SessionEvents.deleted_to_detached` transition is invoked.

Additionally, the :attr:`.InstanceState.persistent` accessor **no longer returns
True** for an object that is in the new "deleted" state; instead, the
:attr:`.InstanceState.deleted` accessor has been enhanced to reliably
report on this new state.   When the object is detached, the :attr:`.InstanceState.deleted`
returns False and the :attr:`.InstanceState.detached` accessor is True
instead.  To determine if an object was deleted either in the current
transaction or in a previous transaction, use the
:attr:`.InstanceState.was_deleted` accessor.

Strong Identity Map is Deprecated
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

One of the inspirations for the new series of transition events was to enable
leak-proof tracking of objects as they move in and out of the identity map,
so that a "strong reference" may be maintained mirroring the object
moving in and out of this map.  With this new capability, there is no longer
any need for the :paramref:`.Session.weak_identity_map` parameter and the
corresponding :class:`.StrongIdentityMap` object.  This option has remained
in SQLAlchemy for many years as the "strong-referencing" behavior used to be
the only behavior available, and many applications were written to assume
this behavior.   It has long been recommended that strong-reference tracking
of objects not be an intrinsic job of the :class:`.Session` and instead
be an application-level construct built as needed by the application; the
new event model allows even the exact behavior of the strong identity map
to be replicated.   See :ref:`session_referencing_behavior` for a new
recipe illustrating how to replace the strong identity map.

:ticket:`2677`

.. _change_3499:

Changes regarding "unhashable" types
------------------------------------

The :class:`.Query` object has a well-known behavior of "deduping"
returned rows that contain at least one ORM-mapped entity (e.g., a
full mapped object, as opposed to individual column values). The
primary purpose of this is so that the handling of entities works
smoothly in conjunction with the identity map, including to
accommodate for the duplicate entities normally represented within
joined eager loading, as well as when joins are used for the purposes
of filtering on additional columns.

This deduplication relies upon the hashability of the elements within
the row.  With the introduction of Postgresql's special types like
:class:`.postgresql.ARRAY`, :class:`.postgresql.HSTORE` and
:class:`.postgresql.JSON`, the experience of types within rows being
unhashable and encountering problems here is more prevalent than
it was previously.

In fact, SQLAlchemy has since version 0.8 included a flag on datatypes that
are noted as "unhashable", however this flag was not used consistently
on built in types.  As described in :ref:`change_3499_postgresql`, this
flag is now set consistently for all of Postgresql's "structural" types.

The "unhashable" flag is also set on the :class:`.NullType` type,
as :class:`.NullType` is used to refer to any expression of unknown
type.

Additionally, the treatment of a so-called "unhashable" type is slightly
different than its been in previous releases; internally we are using
the ``id()`` function to get a "hash value" from these structures, just
as we would any ordinary mapped object.   This replaces the previous
approach which applied a counter to the object.

:ticket:`3499`

.. _change_3321:

Specific checks added for passing mapped classes, instances as SQL literals
---------------------------------------------------------------------------

The typing system now has specific checks for passing of SQLAlchemy
"inspectable" objects in contexts where they would otherwise be handled as
literal values.   Any SQLAlchemy built-in object that is legal to pass as a
SQL value includes a method ``__clause_element__()`` which provides a
valid SQL expression for that object.  For SQLAlchemy objects that
don't provide this, such as mapped classes, mappers, and mapped
instances, a more informative error message is emitted rather than
allowing the DBAPI to receive the object and fail later.  An example
is illustrated below, where a string-based attribute ``User.name`` is
compared to a full instance of ``User()``, rather than against a
string value::

    >>> some_user = User()
    >>> q = s.query(User).filter(User.name == some_user)
    ...
    sqlalchemy.exc.ArgumentError: Object <__main__.User object at 0x103167e90> is not legal as a SQL literal value

The exception is now immediate when the comparison is made between
``User.name == some_user``.  Previously, a comparison like the above
would produce a SQL expression that would only fail once resolved
into a DBAPI execution call; the mapped ``User`` object would
ultimately become a bound parameter that would be rejected by the
DBAPI.

Note that in the above example, the expression fails because
``User.name`` is a string-based (e.g. column oriented) attribute.
The change does *not* impact the usual case of comparing a many-to-one
relationship attribute to an object, which is handled distinctly::

    >>> # Address.user refers to the User mapper, so
    >>> # this is of course still OK!
    >>> q = s.query(Address).filter(Address.user == some_user)


:ticket:`3321`

New Features and Improvements - Core
====================================


.. _change_2528:

A UNION or similar of SELECTs with LIMIT/OFFSET/ORDER BY now parenthesizes the embedded selects
-----------------------------------------------------------------------------------------------

An issue that, like others, was long driven by SQLite's lack of capabilities
has now been enhanced to work on all supporting backends.   We refer to a query that
is a UNION of SELECT statements that themselves contain row-limiting or ordering
features which include LIMIT, OFFSET, and/or ORDER BY::

    (SELECT x FROM table1 ORDER BY y LIMIT 1) UNION
    (SELECT x FROM table2 ORDER BY y LIMIT 2)

The above query requires parenthesis within each sub-select in order to
group the sub-results correctly.  Production of the above statement in
SQLAlchemy Core looks like::

    stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1)
    stmt2 = select([table1.c.x]).order_by(table2.c.y).limit(2)

    stmt = union(stmt1, stmt2)

Previously, the above construct would not produce parenthesization for the
inner SELECT statements, producing a query that fails on all backends.

The above formats will **continue to fail on SQLite**; additionally, the format
that includes ORDER BY but no LIMIT/SELECT will **continue to fail on Oracle**.
This is not a backwards-incompatible change, because the queries fail without
the parentheses as well; with the fix, the queries at least work on all other
databases.

In all cases, in order to produce a UNION of limited SELECT statements that
also works on SQLite and in all cases on Oracle, the
subqueries must be a SELECT of an ALIAS::

    stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1).alias().select()
    stmt2 = select([table2.c.x]).order_by(table2.c.y).limit(2).alias().select()

    stmt = union(stmt1, stmt2)

This workaround works on all SQLAlchemy versions.  In the ORM, it looks like::

    stmt1 = session.query(Model1).order_by(Model1.y).limit(1).subquery().select()
    stmt2 = session.query(Model2).order_by(Model2.y).limit(1).subquery().select()

    stmt = session.query(Model1).from_statement(stmt1.union(stmt2))

The behavior here has many parallels to the "join rewriting" behavior
introduced in SQLAlchemy 0.9 in :ref:`feature_joins_09`; however in this case
we have opted not to add new rewriting behavior to accommodate this
case for SQLite.
The existing rewriting behavior is very complicated already, and the case of
UNIONs with parenthesized SELECT statements is much less common than the
"right-nested-join" use case of that feature.

:ticket:`2528`

.. _change_3516:

Array support added to Core; new ANY and ALL operators
------------------------------------------------------

Along with the enhancements made to the Postgresql :class:`.ARRAY`
type described in :ref:`change_3503`, the base class of :class:`.ARRAY`
itself has been moved to Core in a new class :class:`.types.Array`.

Arrays are part of the SQL standard, as are several array-oriented functions
such as ``array_agg()`` and ``unnest()``.  In support of these constructs
for not just PostgreSQL but also potentially for other array-capable backends
in the future such as DB2, the majority of array logic for SQL expressions
is now in Core.   The :class:`.Array` type still **only works on
Postgresql**, however it can be used directly, supporting special array
use cases such as indexed access, as well as support for the ANY and ALL::

    mytable = Table("mytable", metadata,
            Column("data", Array(Integer, dimensions=2))
        )

    expr = mytable.c.data[5][6]

    expr = mytable.c.data[5].any(12)

In support of ANY and ALL, the :class:`.Array` type retains the same
:meth:`.Array.Comparator.any` and :meth:`.Array.Comparator.all` methods
from the PostgreSQL type, but also exports these operations to new
standalone operator functions :func:`.sql.expression.any_` and
:func:`.sql.expression.all_`.  These two functions work in more
of the traditional SQL way, allowing a right-side expression form such
as::

    from sqlalchemy import any_, all_

    select([mytable]).where(12 == any_(mytable.c.data[5]))

For the PostgreSQL-specific operators "contains", "contained_by", and
"overlaps", one should continue to use the :class:`.postgresql.ARRAY`
type directly, which provides all functionality of the :class:`.Array`
type as well.

The :func:`.sql.expression.any_` and :func:`.sql.expression.all_` operators
are open-ended at the Core level, however their interpretation by backend
databases is limited.  On the Postgresql backend, the two operators
**only accept array values**.  Whereas on the MySQL backend, they
**only accept subquery values**.  On MySQL, one can use an expression
such as::

    from sqlalchemy import any_, all_

    subq = select([mytable.c.value])
    select([mytable]).where(12 > any_(subq))


:ticket:`3516`

.. _change_3132:

New Function features, "WITHIN GROUP", array_agg and set aggregate functions
----------------------------------------------------------------------------

With the new :class:`.Array` type we can also implement a pre-typed
function for the ``array_agg()`` SQL function that returns an array,
which is now available using :class:`.array_agg`::

    from sqlalchemy import func
    stmt = select([func.array_agg(table.c.value)])

A Postgresql element for an aggregate ORDER BY is also added via
:class:`.postgresql.aggregate_order_by`::

    from sqlalchemy.dialects.postgresql import aggregate_order_by
    expr = func.array_agg(aggregate_order_by(table.c.a, table.c.b.desc()))
    stmt = select([expr])

Producing::

    SELECT array_agg(table1.a ORDER BY table1.b DESC) AS array_agg_1 FROM table1

The PG dialect itself also provides an :func:`.postgresql.array_agg` wrapper to
ensure the :class:`.postgresql.ARRAY` type::

    from sqlalchemy.dialects.postgresql import array_agg
    stmt = select([array_agg(table.c.value).contains('foo')])


Additionally, functions like ``percentile_cont()``, ``percentile_disc()``,
``rank()``, ``dense_rank()`` and others that require an ordering via
``WITHIN GROUP (ORDER BY <expr>)`` are now available via the
:meth:`.FunctionElement.within_group` modifier::

    from sqlalchemy import func
    stmt = select([
        department.c.id,
        func.percentile_cont(0.5).within_group(
            department.c.salary.desc()
        )
    ])

The above statement would produce SQL similar to::

  SELECT department.id, percentile_cont(0.5)
  WITHIN GROUP (ORDER BY department.salary DESC)

Placeholders with correct return types are now provided for these functions,
and include :class:`.percentile_cont`, :class:`.percentile_disc`,
:class:`.rank`, :class:`.dense_rank`, :class:`.mode`, :class:`.percent_rank`,
and :class:`.cume_dist`.

:ticket:`3132` :ticket:`1370`

.. _change_2919:

TypeDecorator now works with Enum, Boolean, "schema" types automatically
------------------------------------------------------------------------

The :class:`.SchemaType` types include types such as :class:`.Enum`
and :class:`.Boolean` which, in addition to corresponding to a database
type, also generate either a CHECK constraint or in the case of Postgresql
ENUM a new CREATE TYPE statement, will now work automatically with
:class:`.TypeDecorator` recipes.  Previously, a :class:`.TypeDecorator` for
an :class:`.postgresql.ENUM` had to look like this::

    # old way
    class MyEnum(TypeDecorator, SchemaType):
        impl = postgresql.ENUM('one', 'two', 'three', name='myenum')

        def _set_table(self, table):
            self.impl._set_table(table)

The :class:`.TypeDecorator` now propagates those additional events so it
can be done like any other type::

    # new way
    class MyEnum(TypeDecorator):
        impl = postgresql.ENUM('one', 'two', 'three', name='myenum')


:ticket:`2919`

Key Behavioral Changes - ORM
============================


Key Behavioral Changes - Core
=============================


Dialect Improvements and Changes - Postgresql
=============================================

.. _change_3499_postgresql:

ARRAY and JSON types now correctly specify "unhashable"
-------------------------------------------------------

As described in :ref:`change_3499`, the ORM relies upon being able to
produce a hash function for column values when a query's selected entities
mixes full ORM entities with column expressions.   The ``hashable=False``
flag is now correctly set on all of PG's "data structure" types, including
:class:`.ARRAY` and :class:`.JSON`.  The :class:`.JSONB` and :class:`.HSTORE`
types already included this flag.  For :class:`.ARRAY`,
this is conditional based on the :paramref:`.postgresql.ARRAY.as_tuple`
flag, however it should no longer be necessary to set this flag
in order to have an array value present in a composed ORM row.

.. seealso::

    :ref:`change_3499`

    :ref:`change_3503`

:ticket:`3499`

.. _change_3503:

Correct SQL Types are Established from Indexed Access of ARRAY, JSON, HSTORE
-----------------------------------------------------------------------------

For all three of :class:`~.postgresql.ARRAY`, :class:`~.postgresql.JSON` and :class:`.HSTORE`,
the SQL type assigned to the expression returned by indexed access, e.g.
``col[someindex]``, should be correct in all cases.

This includes:

* The SQL type assigned to indexed access of an :class:`~.postgresql.ARRAY` takes into
  account the number of dimensions configured.   An :class:`~.postgresql.ARRAY` with three
  dimensions will return a SQL expression with a type of :class:`~.postgresql.ARRAY` of
  one less dimension.  Given a column with type ``ARRAY(Integer, dimensions=3)``,
  we can now perform this expression::

      int_expr = col[5][6][7]   # returns an Integer expression object

  Previously, the indexed access to ``col[5]`` would return an expression of
  type :class:`.Integer` where we could no longer perform indexed access
  for the remaining dimensions, unless we used :func:`.cast` or :func:`.type_coerce`.

* The :class:`~.postgresql.JSON` and :class:`~.postgresql.JSONB` types now mirror what Postgresql
  itself does for indexed access.  This means that all indexed access for
  a :class:`~.postgresql.JSON` or :class:`~.postgresql.JSONB` type returns an expression that itself
  is *always* :class:`~.postgresql.JSON` or :class:`~.postgresql.JSONB` itself, unless the
  :attr:`~.postgresql.JSON.Comparator.astext` modifier is used.   This means that whether
  the indexed access of the JSON structure ultimately refers to a string,
  list, number, or other JSON structure, Postgresql always considers it
  to be JSON itself unless it is explicitly cast differently.   Like
  the :class:`~.postgresql.ARRAY` type, this means that it is now straightforward
  to produce JSON expressions with multiple levels of indexed access::

    json_expr = json_col['key1']['attr1'][5]

* The "textual" type that is returned by indexed access of :class:`.HSTORE`
  as well as the "textual" type that is returned by indexed access of
  :class:`~.postgresql.JSON` and :class:`~.postgresql.JSONB` in conjunction with the
  :attr:`~.postgresql.JSON.Comparator.astext` modifier is now configurable; it defaults
  to :class:`.Text` in both cases but can be set to a user-defined
  type using the :paramref:`.postgresql.JSON.astext_type` or
  :paramref:`.postgresql.HSTORE.text_type` parameters.

.. seealso::

  :ref:`change_3503_cast`

:ticket:`3499`
:ticket:`3487`

.. _change_3503_cast:

The JSON cast() operation now requires ``.astext`` is called explicitly
------------------------------------------------------------------------

As part of the changes in :ref:`change_3503`, the workings of the
:meth:`.ColumnElement.cast` operator on :class:`.postgresql.JSON` and
:class:`.postgresql.JSONB` no longer implictly invoke the
:attr:`.JSON.Comparator.astext` modifier; Postgresql's JSON/JSONB types
support CAST operations to each other without the "astext" aspect.

This means that in most cases, an application that was doing this::

    expr = json_col['somekey'].cast(Integer)

Will now need to change to this::

    expr = json_col['somekey'].astext.cast(Integer)



.. _change_3514:

Postgresql JSON "null" is inserted as expected with ORM operations, regardless of column default present
-----------------------------------------------------------------------------------------------------------

The :class:`.JSON` type has a flag :paramref:`.JSON.none_as_null` which
when set to True indicates that the Python value ``None`` should translate
into a SQL NULL rather than a JSON NULL value.  This flag defaults to False,
which means that the column should *never* insert SQL NULL or fall back
to a default unless the :func:`.null` constant were used.  However, this would
fail in the ORM under two circumstances; one is when the column also contained
a default or server_default value, a positive value of ``None`` on the mapped
attribute would still result in the column-level default being triggered,
replacing the ``None`` value::

    obj = MyObject(json_value=None)
    session.add(obj)
    session.commit()   # would fire off default / server_default, not encode "'none'"

The other is when the :meth:`.Session.bulk_insert_mappings`
method were used, ``None`` would be ignored in all cases::

    session.bulk_insert_mappings(
        MyObject,
        [{"json_value": None}])  # would insert SQL NULL and/or trigger defaults

The :class:`.JSON` type now adds a new flag :attr:`.TypeEngine.evaluates_none`
indicating that ``None`` should not be ignored here; it is configured
automatically based on the value of :paramref:`.JSON.none_as_null`.
Thanks to :ticket:`3061`, we can differentiate when the value ``None`` is actively
set by the user versus when it was never set at all.

If the attribute is not set at all, then column level defaults *will*
fire off and/or SQL NULL will be inserted as expected, as was the behavior
previously.  Below, the two variants are illustrated::

    obj = MyObject(json_value=None)
    session.add(obj)
    session.commit()   # *will not* fire off column defaults, will insert JSON 'null'

    obj = MyObject()
    session.add(obj)
    session.commit()   # *will* fire off column defaults, and/or insert SQL NULL

:ticket:`3514`

.. seealso::

  :ref:`change_3514_jsonnull`

.. _change_3514_jsonnull:

New JSON.NULL Constant Added
----------------------------

To ensure that an application can always have full control at the value level
of whether a :class:`.postgresql.JSON` or :class:`.postgresql.JSONB` column
should receive a SQL NULL or JSON ``"null"`` value, the constant
:attr:`.postgresql.JSON.NULL` has been added, which in conjunction with
:func:`.null` can be used to determine fully between SQL NULL and
JSON ``"null"``, regardless of what :paramref:`.JSON.none_as_null` is set
to::

    from sqlalchemy import null
    from sqlalchemy.dialects.postgresql import JSON

    obj1 = MyObject(json_value=null())  # will *always* insert SQL NULL
    obj2 = MyObject(json_value=JSON.NULL)  # will *always* insert JSON string "null"

    session.add_all([obj1, obj2])
    session.commit()

.. seealso::

    :ref:`change_3514`

:ticket:`3514`

Dialect Improvements and Changes - MySQL
=============================================


Dialect Improvements and Changes - SQLite
=============================================


Dialect Improvements and Changes - SQL Server
=============================================

.. _change_3504:

String / varlength types no longer represent "max" explicitly on reflection
---------------------------------------------------------------------------

When reflecting a type such as :class:`.String`, :class:`.Text`, etc.
which includes a length, an "un-lengthed" type under SQL Server would
copy the "length" parameter as the value ``"max"``::

    >>> from sqlalchemy import create_engine, inspect
    >>> engine = create_engine('mssql+pyodbc://scott:tiger@ms_2008', echo=True)
    >>> engine.execute("create table s (x varchar(max), y varbinary(max))")
    >>> insp = inspect(engine)
    >>> for col in insp.get_columns("s"):
    ...     print col['type'].__class__, col['type'].length
    ...
    <class 'sqlalchemy.sql.sqltypes.VARCHAR'> max
    <class 'sqlalchemy.dialects.mssql.base.VARBINARY'> max

The "length" parameter in the base types is expected to be an integer value
or None only; None indicates unbounded length which the SQL Server dialect
interprets as "max".   The fix then is so that these lengths come
out as None, so that the type objects work in non-SQL Server contexts::

    >>> for col in insp.get_columns("s"):
    ...     print col['type'].__class__, col['type'].length
    ...
    <class 'sqlalchemy.sql.sqltypes.VARCHAR'> None
    <class 'sqlalchemy.dialects.mssql.base.VARBINARY'> None

Applications which may have been relying on a direct comparison of the "length"
value to the string "max" should consider the value of ``None`` to mean
the same thing.

:ticket:`3504`

Dialect Improvements and Changes - Oracle
=============================================
