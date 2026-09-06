# Boltdb Driver

This type of driver is already supported by the RoadRunner and does not require any additional installations.

## Configuration

The complete boltdb driver configuration:

{% code title=".rr.yaml" %}

```yaml
version: "3"

kv:
  # User defined name of the storage.
  boltdb:
    # Required section.
    # Should be "boltdb" for the boltdb driver.
    driver: boltdb

    config:
      # Optional section.
      # Default: "rr.db"
      file: "./rr.db"

      # Optional section.
      # Default: 0777
      permissions: 0777

      # Optional section.
      # Default: "rr"
      bucket: "rr"

      # Optional section.
      # Default: 60
      interval: 60
```

{% endcode %}

## Options

Below is a more detailed description of the various boltdb options.:

### File

`file`: Database file path name. In the case that such a file does not exist, RoadRunner will create this file on its
own at startup. Note that this must be an existing directory, otherwise a "The system cannot find the path specified"
error will occur, indicating that the full database pathname is invalid. Might be a full path with
file: `/foo/bar/rr1.db`. Default: `rr.db`.

### Permissions

`permissions`: The file permissions in UNIX format of the database file, set at the time of its creation. If the file
already exists, the permissions will not be changed.

### Bucket

`bucket`: The bucket name. You can create several boltdb connections by specifying different buckets and in this case
the data stored in one bucket will
not intersect with the data stored in the other, even if the database file and other settings are completely
identical.

### Interval

`interval`: The time in seconds between expiration checks. Expired entries can remain readable until the next check.

## Persistence

Values remain in the database after a RoadRunner restart. Expiration timestamps are kept only in memory and are lost at restart. This limitation applies to both v5 and the v6 beta. Reapply expiration timestamps from application data after startup if needed. Use [Redis](./redis.md) when expiration must survive a RoadRunner restart.

Upgrading from v5 to the v6 beta does not require a database format conversion. Stop RoadRunner before backing up the database file.

## Write Errors

In the v6 beta, `kv.Set` and `kv.Delete` return database commit errors through RPC. Handle these errors before treating a write or deletion as successful.
