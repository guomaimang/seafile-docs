## About

- [Docker](https://docker.com/) is an open source project to pack, ship and run any Linux application in a lighter weight, faster container than a traditional virtual machine.

- Docker makes it much easier to deploy [a Seafile server](https://github.com/haiwen/seafile) on your servers and keep it updated.

- The base image configures Seafile with the Seafile team's recommended optimal defaults.

If you are not familiar with docker commands, please refer to [docker documentation](https://docs.docker.com/engine/reference/commandline/cli/).

## For seafile 7.x.x
Starting with 7.0, we have adjusted seafile-docker image to use multiple containers. The old image runs MariaDB-Server、Memcached and Elasticsearch in the same container with Seafile server. Now, we strip the MariaDB-Server、Memcached and Elasticsearch from the Seafile image and run them in their respective containers.

If you plan to deploy seafile 7.0, you should refer to the [Deploy Documentation](https://download.seafile.com/published/support/docker/pro-edition/Deploy%20Seafile-pro%20with%20Docker.md).

If you plan to upgrade 6.3 to 7.0, you can refer to the [Upgrade Documentation](https://download.seafile.com/published/support/docker/pro-edition/6.3%20upgrade%20to%207.0.md).

## For seafile 6.x.x

### Getting Started

Login the Seafile private registry:

```sh
docker login {host}
```

You can find the private registry information on the [customer center download page](https://customer.seafile.com/downloads/)

To run the seafile server container:

```sh
docker run -d --name seafile \
  -e SEAFILE_SERVER_HOSTNAME=seafile.example.com \
  -v /opt/seafile-data:/shared \
  -p 80:80 \
  {host}/seafileltd/seafile-pro:latest
```

Wait for a few minutes for the first time initialization, then visit `http://seafile.example.com` to open Seafile Web UI.

This command will mount folder `/opt/seafile-data` at the local server to the docker instance. You can find logs and other data under this folder.

### Put your licence file

If you have a `seafile-license.txt` licence file, simply put it in the folder `/opt/seafile-data/seafile/`. In your host machine:

```sh
mkdir -p /opt/seafile-data/seafile/
cp /path/to/seafile-license.txt /opt/seafile-data/seafile/
```

Then restart the container.

```sh
docker restart seafile
```

### More configuration Options

#### Custom Admin Username and Password

The default admin account is `me@example.com` and the password is `asecret`. You can use a different password  by setting the container's environment variables:
e.g.

```sh
docker run -d --name seafile \
  -e SEAFILE_SERVER_HOSTNAME=seafile.example.com \
  -e SEAFILE_ADMIN_EMAIL=me@example.com \
  -e SEAFILE_ADMIN_PASSWORD=a_very_secret_password \
  -v /opt/seafile-data:/shared \
  -p 80:80 \
  {host}/seafileltd/seafile-pro:latest
```

If you forget the admin password, you can add a new admin account and then go to the sysadmin panel to reset user password.

#### Let's encrypt SSL certificate

If you set `SEAFILE_SERVER_LETSENCRYPT` to `true`, the container would request a letsencrypt-signed SSL certificate for you automatically.

e.g.

```sh
docker run -d --name seafile \
  -e SEAFILE_SERVER_LETSENCRYPT=true \
  -e SEAFILE_SERVER_HOSTNAME=seafile.example.com \
  -e SEAFILE_ADMIN_EMAIL=me@example.com \
  -e SEAFILE_ADMIN_PASSWORD=a_very_secret_password \
  -v /opt/seafile-data:/shared \
  -p 80:80 \
  -p 443:443 \
  {host}/seafileltd/seafile-pro:latest
```

If you want to use your own SSL certificate:

- create a folder `/opt/seafile-data/ssl`, and put your certificate and private key under the ssl directory.
- Assume your site name is `seafile.example.com`, then your certificate must have the name `seafile.example.com.crt`, and the private key must have the name `seafile.example.com.key`.

#### Modify Seafile Server Configurations

The config files are under `shared/seafile/conf`. You can modify the configurations according to [Seafile manual](https://manual.seafile.com/)

After modification, you need to restart the container:

```sh
docker restart seafile
```

#### Find logs

The seafile logs are under `/shared/logs/seafile` in the docker, or `/opt/seafile-data/logs/seafile` in the server that run the docker.

The system logs are under `/shared/logs/var-log`, or `/opt/seafile-data/logs/var-log` in the server that run the docker.

#### Add a new Admin

Ensure the container is running, then enter this command:

```sh
docker exec -it seafile /opt/seafile/seafile-server-latest/reset-admin.sh
```

Enter the username and password according to the prompts. You now have a new admin account.

### Directory Structure

#### `/shared`

Placeholder spot for shared volumes. You may elect to store certain persistent information outside of a container, in our case we keep various logfiles and upload directory outside. This allows you to rebuild containers easily without losing important information.

- /shared/db: This is the data directory for mysql server
- /shared/seafile: This is the directory for seafile server configuration and data.
- /shared/logs: This is the directory for logs.
  - /shared/logs/var-log: This is the directory that would be mounted as `/var/log` inside the container. For example, you can find the nginx logs in `shared/logs/var-log/nginx/`.
  - /shared/logs/seafile: This is the directory that would contain the log files of seafile server processes. For example, you can find seaf-server logs in `shared/logs/seafile/seafile.log`.
- /shared/ssl: This is directory for certificate, which does not exist by default.

### Upgrading Seafile Server

If you plan to upgrade 6.3 to 7.0, you can refer to the [Upgrade Documentation](https://download.seafile.com/published/support/docker/pro-edition/6.3%20upgrade%20to%207.0.md).

To upgrade to the latest version of seafile 6.3:

```sh
docker pull {host}/seafileltd/seafile-pro:latest
docker rm -f seafile
docker run -d --name seafile \
  -e SEAFILE_SERVER_LETSENCRYPT=true \
  -e SEAFILE_SERVER_HOSTNAME=seafile.example.com \
  -e SEAFILE_ADMIN_EMAIL=me@example.com \
  -e SEAFILE_ADMIN_PASSWORD=a_very_secret_password \
  -v /opt/seafile-data:/shared \
  -p 80:80 \
  -p 443:443 \
  {host}/seafileltd/seafile-pro:latest
```

If you are one of the early users who use the `launcher` script, you should refer to [upgrade from old format](https://github.com/haiwen/seafile-docker/blob/master/upgrade_from_old_format.md) document.

### Backup and Recovery

#### Struct

We assume your seafile volumns path is in `/shared`. And you want to backup to `/backup` directory.
You can create a layout similar to the following in /backup directory:

```struct
/backup
---- databases/  contains database backup files
---- data/  contains backups of the data directory
```

The data files to be backed up:

```struct
/shared/seafile/conf  # configuration files
/shared/seafile/pro-data  # data of es
/shared/seafile/seafile-data # data of seafile
/shared/seafile/seahub-data # data of seahub
```

#### Backup

Steps:

  1. Backup the databases;
  2. Backup the seafile data directory;

[Backup Order: Database First or Data Directory First](../maintain/backup_recovery.md#backup-order-info)

* backing up Database:

  ```bash
  # It's recommended to backup the database to a separate file each time. Don't overwrite older database backups for at least a week.
  cd /backup/databases
  docker exec -it seafile mysqldump  -uroot --opt ccnet_db > ccnet_db.sql
  docker exec -it seafile mysqldump  -uroot --opt seafile_db > seafile_db.sql
  docker exec -it seafile mysqldump  -uroot --opt seahub_db > seahub_db.sql
  ```

* Backing up Seafile library data:

  * To directly copy the whole data directory

    ```bash
    cp -R /shared/seafile /backup/data/
    cd /backup/data && rm -rf ccnet
    ```
  * Use rsync to do incremental backup

    ```bash
    rsync -az /shared/seafile /backup/data/
    cd /backup/data && rm -rf ccnet
    ```

#### Recovery

* Restore the databases:

  ```bash
  cp /backup/data/ccnet_db.sql /shared/ccnet_db.sql
  cp /backup/data/seafile_db.sql /shared/seafile_db.sql
  cp /backup/data/seahub_db.sql /shared/seahub_db.sql
  docker exec -it seafile /bin/sh -c "mysql -uroot ccnet_db < /shared/ccnet_db.sql"
  docker exec -it seafile /bin/sh -c "mysql -uroot seafile_db < /shared/seafile_db.sql"
  docker exec -it seafile /bin/sh -c "mysql -uroot seahub_db < /shared/seahub_db.sql"
  ```

* Restore the seafile data:

  ```bash
  cp -R /backup/data/* /shared/seafile/
  ```

### Troubleshooting

You can run docker commands like "docker exec" to find errors.

```sh
docker exec -it seafile /bin/bash
```
