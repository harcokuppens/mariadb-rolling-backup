## MariaDB Rolling Backup

This repository provides a set of scripts for creating and managing rolling backups
of a MariaDB database. The solution is designed for flexibility, supporting databases
on a local machine, a remote server, or within a Docker container. The main script,
`mariadb-rolling-backup`, orchestrates the entire process, using a special checksum
to optimize storage and ensure data integrity.

### Features

- **Flexible Backup Targets:** Back up MariaDB databases running on localhost, a
  remote host, or within a Docker container.
- **Rolling Backups:** Automatically creates new backups and prunes old ones to keep
  a specified number of unique database states.
- **Space Optimization:** Optional checksum comparison to discard new backups that
  are identical to the previous one, saving significant storage space.
- **Checksum Integrity:** The `mariadb-backup-checksum` script generates a SHA256
  checksum that intelligently ignores irrelevant data like timestamps or logs,
  ensuring that the checksum truly reflects changes to the core data.
- **Data Integrity Verification:** The `mariadb-restore-backup` script can verify the
  integrity of a backup file using its checksum before restoring it, preventing
  restoration from a corrupted file.
- **Docker Integration:** The project includes a `Dockerfile` to build a
  self-contained image, allowing you to run a backup service in a `docker-compose`
  environment.

---

### Installation

1.  **Clone the repository:**

    ```sh
    git clone https://github.com/harcokuppens/mariadb-rolling-backup.git
    cd mariadb-rolling-backup
    ```

2.  **Add scripts to your PATH:** For convenience, you can add the `bin` directory to
    your system's `PATH`.

    ```sh
    export PATH=$PWD/bin:$PATH
    ```

    Alternatively, you can call the scripts directly using their full path.

3.  **Ensure prerequisites are installed:** The scripts require `mariadb-client` to
    be installed on the host machine.

---

### Usage

#### `mariadb-rolling-backup`

This is the main script for performing rolling backups. It creates a new backup of
the specified DATABASE, and based on the settings(options), it cleans up older ones.

```sh
mariadb-rolling-backup [OPTIONS] DATABASE
```

**Options**

- `--user USER_NAME, -u USER_NAME`: The MariaDB user name to connect with (default:
  current system user's name).
- `--password PASSWORD, -p PASSWORD`: The password for the MariaDB user.
- `-H HOSTNAME, --host HOSTNAME`: The hostname of the MariaDB server. If no host or
  container is provided, it defaults to `localhost`.
- `-C CONTAINER, --container CONTAINER`: The name of the Docker container running
  MariaDB. If no host or container is provided, it defaults to `localhost`.
- `-d, --dir DIR`: The directory where backups will be stored (default:
  `./backups/`).
- `-k, --keep NUM`: The number of unique backups to keep (default: 20).
- `-c, --checksum`: Create checksums for backups.
- `-e, --exclude-tables-in-checksum TABLES`: A comma-separated list of tables to
  exclude from the checksum calculation. This is useful for tables with frequently
  changing data like logs or timestamps.
- `-r, --remove-eq-prev`: Removes the previous backup if its checksum is equivalent
  to the new one. This option implies `-c`.
- `-h, --help`: Shows the help message and exits.

**Examples:**

- **Basic daily backup:** The script will create a new backup every time it's run and
  keep up to 20 backups.
  ```sh
  mariadb-rolling-backup mydatabase
  ```
- **Daily backup with space optimization:** This command will create a new backup,
  but if it's identical to the most recent one (based on the checksum), it will
  discard the old backup. The script will only keep a new backup when the database
  state has truly changed.
  ```sh
  mariadb-rolling-backup -r mydatabase
  ```

---

#### `mariadb-backup`

This script handles the core database dump using `mariadb-dump`. It is a helper
script used by `mariadb-rolling-backup`, but can also be used independently.

```sh
mariadb-backup [OPTIONS] DATABASE [FILEPATH]
```

**Options**

- `--user USER_NAME, -u USER_NAME`: The MariaDB user name.
- `--password PASSWORD, -p PASSWORD`: The MariaDB password.
- `-H HOSTNAME, --host HOSTNAME`: The hostname of the MariaDB server.
- `-C CONTAINER, --container CONTAINER`: The name of the Docker container.
- `--no-data`: Excludes table data from the dump, backing up only the database
  schema.
- `-h, --help`: Displays the help message and exits.

**Arguments**

- `DATABASE`: The name of the database to back up.
- `FILEPATH`: The optional path where the dump file will be saved. If not provided,
  it defaults to `./mariadb.sql.gz`.

---

#### `mariadb-restore-backup`

This script restores a MariaDB database from a gzipped dump file. It can perform an
integrity check using a checksum file if one exists.

```sh
mariadb-restore-backup [OPTIONS] DATABASE FILEPATH
```

**Options**

- `--user USER_NAME, -u USER_NAME`: The MariaDB user name.
- `--password PASSWORD, -p PASSWORD`: The MariaDB password.
- `-H HOSTNAME, --host HOSTNAME`: The hostname of the MariaDB server.
- `-C CONTAINER, --container CONTAINER`: The name of the Docker container.
- `-h, --help`: Displays the help message and exits.

**Arguments**

- `DATABASE`: The name of the database to restore.
- `FILEPATH`: The path to the database dump file (`.sql.gz`).

**Important:** Before restoring, the script checks for a checksum file (`.sha256`)
and verifies its integrity. If the checksums don't match, it will not proceed with
the restoration.

---

#### `mariadb-backup-checksum`

This script computes a unique SHA256 checksum for a MariaDB dump file. It's designed
to ignore specific tables, which is crucial for the space-saving functionality of
`mariadb-rolling-backup`.

```sh
mariadb-backup-checksum [OPTIONS] FILEPATH
```

**Options**

- `-e, --exclude-tables-in-checksum TABLES`: A comma-separated list of tables to
  exclude from the checksum calculation.
- `-h, --help`: Displays the help message and exits.

**Arguments**

- `FILEPATH`: The path to the database dump file.

---

### Docker Usage

You can use the provided `Dockerfile` to create a dedicated backup service. This is
particularly useful in `docker-compose` setups.

1.  **Build the Docker image:**

    ```sh
    docker build -t mariadb-rolling-backup .
    ```

2.  **Integrate with `docker-compose`:** Add the `mariadb-rolling-backup` service to
    your `docker-compose.yml` file. This example shows how to back up a service named
    `db` every night at 2:00 AM.

    ```yaml
    networks:
      mynetwork:
        name: mynetwork

    services:
      mariadb:
        image: mariadb:${MARIADB_VERSION}
        container_name: mariadb
        hostname: mariadb
        restart: unless-stopped
        networks:
          - mynetwork
        environment:
          - MYSQL_ROOT_PASSWORD=rootpw
          - MYSQL_USER=myuser
          - MYSQL_PASSWORD=mypw
          - MYSQL_DATABASE=mydb
        command: --max-connections=1000 --max-allowed-packet=1G
        volumes:
          - ./data/mariadb:/var/lib/mysql
        healthcheck:
          test:
            [
              "CMD",
              "mariadb-admin",
              "ping",
              "-h",
              "localhost",
              "-u",
              "root",
              "-prootpw",
            ]
          interval: 10s
          timeout: 5s
          retries: 5

      backup:
        image: mariadb-rolling-backup
        container_name: mariadb-rolling-backup
        hostname: backup
        restart: unless-stopped
        networks:
          - mynetwork
        depends_on:
        mariadb:
          condition: service_healthy
        environment:
          # These variables are used to access the mariadb database
          - CONTAINER_TIMEZONE=Europe/Amsterdam
          - MYSQL_HOST=mariadb
          - MYSQL_ROOT_PASSWORD=rootpw
          - MYSQL_DATABASE=mydb
          # below variables are used to generate the crontab entry for the mariadb-rolling-backup command
          - CRON_SCHEDULE=30 0 * * *
          - MAX_NUM_BACKUPS_TO_KEEP=20 # -k VALUE option
          - CREATE_CHECKSUMS=true # -c --checksum option (if true )
          # comma separated list of tables to exclude from checksum calculation, e.g. "table1,table2"
          - TABLES_TO_EXCLUDE_IN_CHECKSUM="" # -e --exclude-tables-in-checksum option
          # if true, will discard a backup if its checksum is the same as the previous backup
          - REMOVE_CHECKSUM_EQUIV_PREVIOUS_BACKUP=true # -r --remove-eq-prev option
          # note:REMOVE_CHECKSUM_EQUIV_PREVIOUS_BACKUP=true implies CREATE_CHECKSUMS=true
          # in the container the backups are always at /backups; but you can set a different location for the logfile
          - LOGFILE=/backups/backup.log
        volumes:
          # location to store backups on the host; in the container the backups are at /backups
          - ./backups:/backups
    ```

This setup ensures that a backup is run periodically and that it's correctly placed
in the `backups` directory on your host machine.
