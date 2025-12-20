# Share Your Data Using Couchdb Replication

{% hint style="warning" %}
**Experimental Feature**

This feature may be removed or changed in the future. **It might also have unhandled cases and cause corruption in your data. Be sure to backup your data constantly while using this.**
{% endhint %}

{% hint style="info" %}
**Advanced Functionality**

You're recommended to have some background knowledge about CouchDB before doing this.
{% endhint %}

Thanks to the power of CouchDB, sharing your whole database or parts of your data with another user can be easily done with CouchDB Replication, with the following use cases supported:

* Share items only in specific collections.
* Share items by custom rules.
* Share with read-only access.
* Share with read-write access.

However, it has the following limitations:

* One user must give their CouchDB database access to the other user, which means it's possible that a user can see or change everything in the other user's database.
* The people you shared your data with won't be able to change the Reference Number of your collection and items.
* If you set a Two-way Replication (Read-write Sharing), the person with whom you share your data with might be able to mess up with your unshared data if they know it's ID.

## Prerequisite

Both you and the one you want to share data with must have CouchDB sync setup. The two CouchDB databases aren't required to be on the same server, but they should be reachable to one another.

## Setup a One-way Replication - For Read-only Sharing

You'll need to create a replication document in the `_replicator` database on one of the servers. Here's a sample:

<pre class="language-json"><code class="lang-json"><strong>{
</strong>  "_id": "an_unique_id_for_your_replication",
  "source": {
    "url": "http://127.0.0.1:5984/soruce_database_name",
    "headers": {
      "Authorization": "Basic dXNlcm5hbWU6cGFzc3dvcmQ="
    }
  },
  "target": {
    "url": "https://127.0.0.1/target_database_name",
    "headers": {
      "Authorization": "Basic ZXhhbXBsZTpwYXNzd29yZA=="
    }
  },
  "continuous": true,
  "selector": {
    "type": { "$or": [{ "type": "collection" }, { "type": "item" }] }
  }
}
</code></pre>

You can copy the JSON document above and paste it into the editor in `http(s)://<your_couchdb_server>:<port>/_utils/#database/_replicator/_new` (e.g. [http://127.0.0.1:5984/\_utils/#database/\_replicator/\_new](http://127.0.0.1:5984/\_utils/#database/\_replicator/\_new)).

Note that you'll need to change some of the fields in your document:

* `_id`: You can choose a recognizable ID for your replication. It should be unique.
* `source`: Set the connection of the source database, i.e., the database storing the items to be shared.
  * For the Authorization field, encode the username and password as `username:password` with Base64. For example, by running `print -n 'example:password' | base64` in \*nix.
* `target`: Set the connection of the target database, i.e., the database that you want to share the items with.
* `selector`: Use this to share every collection and item or specific collections and items. See [examples below](share-your-data-using-couchdb-replication.md#selector-examples). **This MUST be set to at least `{ "$or": [{ "type": "collection" }, { "type": "item" }] }` to prevent overwriting everything in the target database.** For more info, see [https://docs.couchdb.org/en/stable/api/database/find.html#selector-syntax](https://docs.couchdb.org/en/stable/api/database/find.html#selector-syntax).

For more info, see [https://docs.couchdb.org/en/stable/json-structure.html#replication-settings](https://docs.couchdb.org/en/stable/json-structure.html#replication-settings).

After saving the replication, you can see its status at `http(s)://<your_couchdb_server>:<port>/_utils/#/replication` (e.g. [http://127.0.0.1:5984/\_utils/#/replication](http://127.0.0.1:5984/\_utils/#/replication)).

### Selector Examples

<details>

<summary>Replicate every collection and item</summary>

```json
{ "$or": [{ "type": "collection" }, { "type": "item" }] }
```

</details>

<details>

<summary>Replicate specific collections, and items in those collecitons</summary>

```json
{
  "$or": [
    {
      "type": "collection",
      "_id": {
        "$in": [
          "collection-<collection_id_1>",
          "collection-<collection_id_2>"
        ]
      }
    },
    {
      "type": "item",
      "data.collection_id": {
        "$in": [
          "<collection_id_1>",
          "<collection_id_2>"
        ]
      }
    },
    {
      "type": "item_image",
      "data._item_collection_id": {
        "$in": [
          "<collection_id_1>",
          "<collection_id_2>"
        ]
      }
    },
    {
      "type": "image",
      "data._item_collection_ids": {
        "$elemMatch": {
          "$in": [
            "<collection_id_1>",
            "<collection_id_2>"
          ]
        }
      }
    }
  ]
}
```

Note that you'll need to include the `collection-` prefix in the collection IDs under `"type": "collection"`.

</details>

[https://docs.couchdb.org/en/stable/replication/intro.html#controlling-which-documents-to-replicate](https://docs.couchdb.org/en/stable/replication/intro.html#controlling-which-documents-to-replicate) asdf

## Setup a Two-way Replication - For Read-write Sharing

1. Create a replication document the same as [Setup a One-way Replication](share-your-data-using-couchdb-replication.md#setup-a-one-way-replication-for-read-only-sharing).
2. Create another replication document with the same content as step 1 but with the `source` and `target` databases swapped.
3. Ask the person you want to share data with for their _Configuration UUID_. The _Configuration UUID_ can be found in the app (More -> Settings -> Configurations).
4. Create a document in your source database with `_id` set to `db_sharing-<your_config_uuid>--<their_config_uuid>`, as follows:

```json
{
  "_id": "db_sharing-<your_config_uuid>--<their_config_uuid>",
  "data": {
    "permissions": ["read", "write"]
  }
}
```

Note: This document will not actually permit one to write into your database. It will just unblock the UI for editing.

5. Edit the replication document you created on step 1, and add the ID of step 4 to the `selector` section. For example:

```diff
   "selector": {
     "$or": [
+      { "_id": "db_sharing-<your_config_uuid>--<their_config_uuid>" },
       {
         "type": "collection",
         "_id": {
           "$in": ["collection-..."]
         }
       },
       {
         "type": "item",
         "data.collection_id": {
           "$in": ["..."]
         }
       }
     ]
   },
```

## How to stop sharing?

1. Delete the `db_sharing-<your_config_uuid>--<their_config_uuid>` document in your database, if you had created one.
2. Remove the corresponding replications from the `_replicator` database.
3. Optionally, ask the person you shared data with to [purge](https://docs.couchdb.org/en/latest/api/database/misc.html#db-purge) the documents that came from your database from their own database. Note: do not use delete, since the deletion will be replicated back to your database if you do replication from their database again.

## Troubleshooting

<details>

<summary>Got error <code>Save failed: Document update conflict.</code> on creating a new replication document</summary>

Check if you have the `_rev` field in your new replication document. If so, delete it.

</details>
