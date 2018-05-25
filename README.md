# Field Resize Example

In this tutorial, we will see how to increase the size of an existing _Text
(Plain)_ field in Drupal 8 without losing data using a `hook_update_N()`.

## The problem

Drupal's field system is great, however, there are certain things which could
be improved. Say, you have a _Text (Plain)_ field named `field_one_liner` which
is 64 characters long. You created around 15 nodes and then you realized that
the field size should have been 255. If you try to do this from Drupal's field
management UI, you will get a message saying:

>There is data for this field in the database. The field settings can no longer
be changed.
 
So, the only way you can resize it is after deleting the existing field! This
doesn't make much sense because it is indeed possible to increase field's size
without losing data.

## Assumptions

It has been assumed that:

* You have intermediate /advanced knowledge of Drupal.
* You know how to develop modules for Drupal.

## Prerequisites

If you're going to try out the code provided in this example, make sure you
have the following field on any node type:

* Name: One-liner
* Machine name: `field_one_liner`
* Type: Text (Plain)
* Length: 64

After you configure the field, create some nodes with some data on the
_One-liner_ field.

**Note:** Reducing the length of a field might result in data loss.

## Implementing hook_update_N()

`hook_update_N()` lets you run commands to update the database schema. You can
create, update and delete database tables and columns using this hook after
your module has been installed. To implement this hook, you need to have a
custom module. For this example, I've implemented this hook in a custom module
which I've named `custom_field_resize`.

```php
/**
 * Increase the length of "field_one_liner" to 255 characters.
 */
function custom_field_resize_update_8001() {
}
``` 

To change the field size, there are four things we will do inside this hook.

### Resize the Columns

We will run a set of queries to update the relevant database columns.

```php
$database = \Drupal::database();
$database->query("ALTER TABLE node__field_one_liner MODIFY field_one_liner_value VARCHAR(255)");
$database->query("ALTER TABLE node_revision__field_one_liner MODIFY field_one_liner_value VARCHAR(255)");
```

If revisions are disabled then the `node__field_one_liner` table won't exist.
So, you can remove the second query if your entity doesn't allow revisions.

### Update Storage Schema

Resizing the columns with a query is not sufficient. Drupal maintains a record
of what database schema is currently installed. If we don't do this then Drupal
will think that the database schema needs to be updated because the column
lengths in the database and will not match the configuration storage. 

```php
$storage_key = 'node.field_schema_data.field_one_liner';
$storage_schema = \Drupal::keyValue('entity.storage_schema.sql');
$field_schema = $storage_schema->get($storage_key);
$field_schema['node__field_one_liner']['fields']['field_one_liner_value']['length'] = 255;
$field_schema['node_revision__field_one_liner']['fields']['field_one_liner_value']['length'] = 255;
$storage_schema->set($storage_key, $field_schema);
```

The above code will update the `key_value` table to store the updated length
of the `field_one_liner` in it's configuration.

### Update Field Configuration

We took care of the database schema data. However, there are other places where
Drupal stores the configuration. Now, we will need to tell the Drupal config
management system that the field length is 255. 

```php
// Update field configuration.
$config = \Drupal::configFactory()
  ->getEditable('field.storage.node.field_one_liner');
$config->set('settings.max_length', 255);
$config->save(TRUE);
```

Finally, Drupal also stores info about the actively installed configuration and
schema. To refresh this, we will need to resave the field storage configuration
to make Drupal detect all our changes.

```php
// Update field storage configuration.
FieldStorageConfig::loadByName($entity_type, $field_name)->save();
```

After this, running `drush updb` or running `update.php` from the admin
interface should detect your `hook_update_N()` and it should update your field
size. If you're committing your configuration to git, you'll need to run
`drush config-export` after running the database updates to update the config
in the filesystem and then commit it.

## Conclusion

Though we've talked about resizing a _Text (Plain)_ or _varchar_ field in this
tutorial, we can do the same for any field type which can be safely resized
using SQL. In certain complex scenarios, it might be necessary to create a
temporary table with the new data-structure, copy the existing data into that
table with queries and once all the data has been copied successfully, replace
the existing the table with the temporary table.

Maybe someday we'll have this resizing feature in Drupal where Drupal will
intelligently allow us to increase a field's size from it's field UI and only
deny reduction of field size where there is a possibility of data loss.
