1. Start the primary container: `docker compose up -d primary`
1. Access the primary container with `docker exec -it primary /bin/bash` to execute the instructions below:
    1. Set owner of the WAL-G storage directory to postgres: `chown -R postgres:postgres /wal-g-storage`
    1. Switch user to postgres: `su - postgres`
    1. Make sure user postgres has explicit replication rights and insert some test data with psql: `psql -c "alter user postgres replication;create table test (value text primary key);insert into test values ('test1');"`
    1. Create a WAL-G backup: `wal-g --config /wal-g.json backup-push /var/lib/postgresql/data`
    1. Exit the container (e.g. with CTRL+D) and kill the Compose stack: `docker compose down`
1. Remove the volume with primary's data directory: `docker volume rm walg_replica_repro_primary-data`
1. Fetch the backup in primary: `docker run -v $HOME/walg_replica_repro/primary-data:/data -v $HOME/walg_replica_repro/wal-g.json:/wal-g.json -v $HOME/walg_replica_repro/wal-g-storage:/wal-g-storage postgres:latest wal-g --config /wal-g.json backup-fetch /data ${REPLACE_WITH_BACKUP_NAME}`
1. Fetch the backup in replica: `docker run -v $HOME/walg_replica_repro/replica-data:/data -v $HOME/walg_replica_repro/wal-g.json:/wal-g.json -v $HOME/walg_replica_repro/wal-g-storage:/wal-g-storage postgres:latest wal-g --config /wal-g.json backup-fetch /data ${REPLACE_WITH_BACKUP_NAME}`
