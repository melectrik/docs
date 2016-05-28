Storing Data
==============

ViUR comes with its own, unique data model access API.

The concept behind these models is similar to the one used within the Google AppEngine™ in general.

However, ViURs implementation of the data model layer is much more powerful:

- It also stores meta data (descriptions for each field, help and error texts, visible and editable to/by the use etc.)
- More fine-grained. E.g. Numeric Bones allows specification of min and max values and its precision
- Complex types: relations, captchas, text documents

And best of all: Its entirely extensible! If you need a complex type that is not part of the system, its easy to add it on your own.
The application developer has two ways accessing the database: Low and highlevel.

The lowlevel approach is provided by server.db. This allows querying and storing data without the need to define models
first. Its a thin proxy to the object-storage provided by the datastore. If low-level access is needed, the application
*must* use this modul. Its not valid to use the ext.db or ext.ndb modules supplied by the appengine directly. To
enhance the speed and lower the costs of database queries, ViUR uses the memcache to efficiently serve queries where possible.
Altering data directly through one of these APIs will lead to cache-inconsistency and might result in stale data being delivered
to clients. The highlevel API allows querying along a defined model. This allows easy access to complex queries (i.e. filtering by
an relational property), which are not directly possible on the datastore.

Lowlevel API
-------------

Creating an Entity
^^^^^^^^^^^^^^^^^^^

To store a new entity in the datastore, first create an instance of db.Entity.
An kind must be specified. This is the collection (think of an SQL-Tablename) for that entity.
::

	dbObj = db.Entity("person")


Then its possible to add data to this entity.

::

	dbObj["name"] = "Alice"
	dbObj["age"] = 42

Finally save that entry to the datastore.

::

	db.Put(dbObj)


After saving the entity, its valid to call dbObj.key() to retrieve the key under which it has been saved.

Querying Data
^^^^^^^^^^^^^
To start a query, create a instance of db.Query.

::

	query = db.Query("person")

After creating a query for a given kind, its possible to filter / sort the results.

.. Warning::
	The low-level API does **not** prevent you from running invalid queries.
	Filtering and ordering works only for indexed properties. If you filter or sort by an property that doesn't exist
	or is not indexed the query will **silently** fail and return zero results.

The following will filter the results by age and sort these results from Z to a.

::

	query.filter("age =", 20).order(("name",query.DESCENDING))


To fetch these results, use query.run(). If you only need one result (the first one if there are more), use query.get().
Query.run() will always return an iterator yielding db.Entity, query.get() will return one db.Entity if there is at least one result,
None otherwise.

If an Entity is not needed anymore, you can delete it using db.Delete()
::

	db.Delete(entity.key())


ViURs database API provides more functions than these basic ones; see the API reference for more details.

Highlevel API
-------------

The highlevel-api operates on our bones/skeleton system.
This allows complex queries (ie. relations) and provides a safety net regarding invalid queries.


Defining the Model
^^^^^^^^^^^^^^^^^^

As the highlevel API operates on a datamodel, its required to define such a model first.

::

	class Person(Skeleton):
		kindName = "person"
		name = stringBone(descr="Name", required=True, indexed=True)
		age = numericBone(descr="Age", required=True, indexed=True,
					min=0, max=150, precision=0)


This will define a Model called Person with two properties (name and age). Both properties are required
(ie. the user is forced to supply valid data for these properties before he's able to save it),
and all entities are saved as kind *person*.
For stringBones, required=True means a non-empty string; for the numericBone a value between min and max (inclusive).
Detailed information about all included bones, and their possible parameters can be found in :ref:`server-bones-datatypes`.

Storing data
^^^^^^^^^^^^

You create an entity by calling its corresponding skeleton-class.
Unlike the lowlevel API, its only possible to store data according to the defined datamodel.
Its possible (but not recommend, as its possible to assign invalid data) to directly set the data of individual bones.

::

	skel = Person()
	skel["name"].value = "Alice"
	skel["age"].value = 42


The recommended method is calling :py:func:`server.skeleton.Skeleton.fromClient` on the skeleton.
This tries to parse the data according its model, and returns True on success (meaning the given data was successful parsed for this model), False otherwise.
Even if this function returns False, all bones are guaranteed to be in a valid state; the ones which have been read correctly contain their data,
the other ones are set back to a safe default (None in most cases).
So its possible to call toDB() afterwards even if reading data fromClient failed (through this might violate your
assumptions about required fields!).

.. Note::

	This function never alters the dictionary passed to this function. Its safe to supply user-supplied data
	to this function.

::

	skel = Person()
	success = skel.fromClient( { "name": "Alice", "age": 42 } )


After providing data according to the model, safe its data using skel.toDB()

::

	key = skel.toDB()

This function returns the key (as string) the entity has been saved as.
If you want to update an existing entity, you have to load the old one first using skel.fromDB().


.. Note::
	If you used the low-level API to store additional values in that entity that have no representation in your
	skeleton-model, these values are left unchanged by skel.toDB(). toDB() only updates the fields known to the
	skeleton; it will never touch fields unknown to it.

Querying
^^^^^^^^

First, create an query object calling :py:func:`server.skeleton.Skeleton.all` on a skeleton.

::

	query = Person.all()

This returns a db.Query instance, bound to this model.
All functions mentioned in the lowlevel API are valid on this object, too. However, there are additional Methods available.
:py:func:`server.db.mergeExternalFilter` allows querying along the datamodel.
This function is aware of our extended syntax, allowing querying relations. This syntax is equal to the syntax used in
HTTP-Requests.

::

	query.mergeExternalFilter({"age": 20, "orderby": "name", "orderdir": "1"})

.. Note::
	Its safe to use client-supplied data in this function, as invalid parameters are ignored.
	This function also warns you if an unsatisfiable query is supplied.


After all filters and orders are applied, the results can be fetched using :py:func:`server.db.Query.run` or :py:func:`server.db.Query.get`.
If you want a skeleton (or a skellist) instead of the raw db.Entities, call query.getSkel() or query.fetch().

::

	skel = query.getSkel()

If not needed any more, delete an entity by calling :py:func:`server.skeleton.Skeleton.delete` on your skeleton.

::

	skel.delete()

.. Warning::
	If an Entity has been created/modified using the highlevel API, always delete it using the highlevel-API.
	Otherwise there will be garbage left in the datastore.

