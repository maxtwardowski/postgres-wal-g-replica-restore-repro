# Issue reproduction

We ran into an issue where the primary and replica, restored concurrently from the same backup, fail to follow the same timeline and end up stuck with a variety of errors, suggesting that replica is ahead of the primary.

Run the following steps to reproduce this:

## 1. Create the initial backup

Some of this could be simplified, but we're making all the steps more explicit to replicate what is happening in our codebase.

```bash
# 1. Initialize the primary database
docker run --rm \
    -v $(pwd)/primary-data:/data \
    postgres:14.5 \
    /bin/bash -c "chown -R postgres:postgres /data; su postgres -c \"initdb -D /data --username=postgres\";"
    
# 2. Append `include_dir`, archive directives to the config (`sudo` just to be sure :))
sudo sh -c "echo \"include_dir = '/custom-pg-configs'\" >> ./primary-data/postgresql.conf"
sudo sh -c "echo \"archive_mode = on\" >> ./primary-data/postgresql.conf"
sudo sh -c "echo \"archive_command = '/usr/local/bin/wal-g --config /wal-g.json wal-push %p'\" >> ./primary-data/postgresql.conf"

# 3. Start the primary container
docker compose up -d primary

# 4. Enter the container
docker exec -it primary /bin/bash

# Then, in the container, run the following commands

# 4a. Change the owner of the WAL-G storage directory to `postgres`
chown -R postgres:postgres /wal-g-storage

# 4b. Switch to the `postgres` user 
su - postgres

# 4c. Give `postgres` replication rights, insert some test data
psql -c "alter user postgres replication;"
psql -c "alter user postgres encrypted password 'postgres';"
psql -c "create table test (value text primary key);"
psql -c "insert into test values ('test1');"

# 4d. Create a WAL-G backup
wal-g --config /wal-g.json backup-push /data

# 4e. Exit the container
exit # exit to the root shell
exit # exit to your shell

# 4f. Stop the database, kill the containers
docker compose stop primary
# This should happen pretty quickly, but you can ensure that it's done stopping in `docker logs primary`.
docker compose down

# 5. Remove the primary's data directory
sudo rm -rf ./primary-data

# 6. Find the backup name
sudo ls ./wal-g-storage/basebackups_005
export BACKUP_NAME="base_000000010000000000000002"

# 6. Fetch the backup as the primary
docker run --rm \
    -v $(pwd)/primary-data:/data \
    -v $(pwd)/wal-g.json:/wal-g.json \
    -v $(pwd)/wal-g-storage:/wal-g-storage \
    postgres_repro:latest \
    wal-g \
        --config /wal-g.json \
        backup-fetch \
        /data \
        ${BACKUP_NAME}

# 7. Fetch the backup as the replica
docker run --rm \
    -v $(pwd)/replica-data:/data \
    -v $(pwd)/wal-g.json:/wal-g.json \
    -v $(pwd)/wal-g-storage:/wal-g-storage \
    postgres_repro:latest \
    wal-g \
        --config /wal-g.json \
        backup-fetch \
        /data \
        ${BACKUP_NAME}

# 8. Prepare primary's and replica's configs for restoring from backup
sudo cp ./primary_restore.conf ./primary-config/primary_restore.conf
mkdir -p ./replica-config
cp ./replica_restore.conf ./replica-config/replica_restore.conf

# 9. Create primary and standby signals
sudo touch ./primary-data/recovery.signal
sudo touch ./replica-data/standby.signal

# 10. Add recovery_command to the primary's and replica's config
sudo sh -c "echo \"restore_command = '/usr/local/bin/wal-g --config /wal-g.json wal-fetch %f %p'\" >> ./primary-data/postgresql.conf"
sudo sh -c "echo \"restore_command = '/usr/local/bin/wal-g --config /wal-g.json wal-fetch %f %p'\" >> ./replica-data/postgresql.conf"

# 11. Allow replication from replica in pg_hba.conf
sudo sh -c "echo \"host all all 0.0.0.0/0 scram-sha-256\nhost replication all 0.0.0.0/0 scram-sha-256\" >> ./primary-data/pg_hba.conf"

# 12. Spin up both databases at once
docker compose up -d

# 13. Create the replication slot on primary
docker exec -it --user "postgres" primary /bin/bash -c "psql -c \"select pg_create_physical_replication_slot('replica');\""
```

1. Start primary in detached mode: `dc up -d primary`
1. Start replica: `dc up replica`. The database will shutdown as soon as the recovery is complete.
1. Remove the replica_restore.conf: `rm ./replica-config/replica_restore.conf`
1. Copy replica_standby.conf: `cp ./replica_standby.conf ./replica-config/replica_standby.conf`
1. Create standby.signal file for replica: `touch ./replica-data/standby.signal`