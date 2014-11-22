=================================
IID Generator Using Elasticsearch
=================================

At our company we are using `Crate <http://crate.io>`_ as our database. Crate
has no built in support for autoincrementing integer ids.

So I came up to build my own generator:

    - needs to be fast (so not really superfast)
    - needs to work on a distributed system

First I found this `blog post from Clinton Gormley <http://blogs.perl.org/users/clinton_gormley/2011/10/elasticsearchsequence---a-blazing-fast-ticket-server.html>`_
expaining how to use the version numbering built into elasticsearch to create
ids. The use is really simple but it has a big disadvantage, it can not be
backuped useing simple bulk files because it is not possible to write the
_version back into elasticsearch.

The current iid value needs to be stored in the _source of the document to be
able to dump the data.


Generator Explained (curl)
==========================

First I create a new document containing an integer value::

    >>> curl -XPUT http://localhost:9200/sequence/sequence/1?pretty= -d '{"iid": 0}'
    {
        "_index" : "sequence",
        "_type" : "sequence",
        "_id" : "1",
        "_version" : 1,
        "created" : true
    }

First I need to increment the current id. This can be done without the need of
a read/modify/write cycle of the document by using the _update API::

    >>> curl -XPOST http://localhost:9200/sequence/sequence/1/_update -d '
    ... {
    ...     "script": "ctx._source.iid += 1",
    ...     "lang": "groovy"
    ... }
    ... '
    {
        "_index" : "sequence",
        "_type" : "sequence",
        "_id" : "1",
        "_version" : 2
    }

The response is not returning the changed document. With the parameter
`fields` the fields to be returns can be specified::

    >>> curl -XPOST http://localhost:9200/sequence/sequence/1/_update?fields=iid -d '
    ... {
    ...     "script": "ctx._source.iid += 1",
    ...     "lang": "groovy"
    ... }
    ... '
    {
        "_index" : "sequence",
        "_type" : "sequence",
        "_id" : "1",
        "_version" : 3,
        "get" : {
            "found" : true,
            "fields" : {
                "iid" : [ 2 ]
            }
        }
    }

All this works but it is not performing very good because I need a request
for every id. But it is also possible to request a bulk of ids by incrementing
the id by more than just 1::

    >>> curl -XPOST http://localhost:9200/sequence/sequence/1/_update?fields=iid -d '
    ... {
    ...     "script": "ctx._source.iid += bulk_size",
    ...     "params": {"bulk_size": 10},
    ...     "lang": "groovy"
    ... }
    ... '
    {
        "_index" : "sequence",
        "_type" : "sequence",
        "_id" : "1",
        "_version" : 4,
        "get" : {
            "found" : true,
            "fields" : {
                "iid" : [ 12 ]
            }
        }
    }

Now I have requested 10 ids in one request. The ids I am allowed to use are
in the range(iid-10+1, iid).

Also note how I used the script this time. I use the `params` to define the
`bulk size`. This has the advantage that the script code is always the same.
elasticsearch can now use the cached version of the script and work faster
because it needs to compile the code only once.

The disadvantage in using bulks is that it is possible to create holes if an
app is restarted which has not fully consumed the reuested bulk.


Optimized for real world use
============================

First I configure the index with a mapping. This needs to be done before the
first write to the index::

    >>> curl -XPOST http://localhost:9200/sequence/_mapping -d '
    ... {
    ...     "settings": {
    ...         "number_of_shards": 1,
    ...         "auto_expand_replicas": "0-all"
    ...     },
    ...     "mappings": {
    ...         "sequence": {
    ...             "_all": {"enabled": 0},
    ...             "_type": {"index": "no"},
    ...             "dynamic": "strict",
    ...             "properties": {
    ...                 "iid": {
    ...                     "type": "string",
    ...                     "index": "no",
    ...                 },
    ...             },
    ...         }
    ...     }
    ... }'

Then I optimize the update request::

    >>> curl -XPOST "http://localhost:9200/sequence/sequence/1/_update?fields=iid&retry_on_conflict=5" -d '
    ... {
    ...     "script": "ctx._source.iid += bulk_size",
    ...     "params": {"bulk_size": 10},
    ...     "lang": "groovy",
    ...     "upsert": {
    ...         'iid': 10
    ...     },
    ... }'

`upsert` will be used if the document doesn't exists to create the document
with the initial data. The value for `iid` must always be set to the
bulk_size.

The query parameter `retry_on_conflict` will retry the update if there is a
version conflict from the time the document is read until it is updated.


With Python
===========

This is how it can be used with the `elasticsearch python client <http://elasticsearch-py.readthedocs.org/>`_.

Get a client instance::

    >>> from elasticsearch import Elasticsearch
    >>> es = Elasticsearch()

Create the mapping::

    >>> es.indices.create(
    ...     'sequence',
    ...     {
    ...         "settings": {
    ...             "number_of_shards": 1,
    ...             "auto_expand_replicas": "0-all"
    ...         },
    ...         "mappings": {
    ...             "sequence": {
    ...                 "_all": {"enabled": 0},
    ...                 "_type": {"index": "no"},
    ...                 "dynamic": "strict",
    ...                 "properties": {
    ...                     "iid": {
    ...                         "type": "string",
    ...                         "index": "no",
    ...                     },
    ...                 },
    ...             }
    ...         }
    ...     },
    ...     ignore=400  # ignore index already exists
    ... )

Request a bulk::

    >>> bulk_size = 10
    >>> result = self._es_client().update(
    ...     index='lc_iidsequences',
    ...     doc_type='iid',
    ...     id=self.name,
    ...     body={
    ...         "script": "ctx._source['iid'] += bulk_size",
    ...         "lang": "groovy",
    ...         "params": {
    ...             "bulk_size": bulk_size
    ...         },
    ...         "upsert": {
    ...             'iid': bulk_size
    ...         },
    ...     },
    ...     retry_on_conflict=10,
    ...     fields='iid',
    ... )
    >>> iid = result['get']['fields']['iid'][0]
    >>> bulk = range(iid, iid - self.bulk_size, -1)

Now the ids can be retrieved from `bulk` until it is exhausted::

    >>> id = bulk.pop()


References
==========

    - `elasticsearch _update API <http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/docs-update.html>`_
    - `elasticsearch _mapping API <http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/indices-put-mapping.html>`_
