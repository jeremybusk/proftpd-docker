# proftpd-docker

## Requirements

- docker
- (docker-compose)
- PostgreSQL instance
- Creating the necessary tables on the PostgreSQL instance using the included migration: `proftp_tables.sql`.
- openssl (for creating passwords)

## Running with docker-compose

* Create a `.env` containing the requirements environnement variables

This file should be located next to the provided `docker-compose.yml` file.
The `.env.tpl` file can be used to bootstrap the `.env` file.

The required/optional parameters are described here after:

- **FTP_DB_HOST**: db hostname or ip address, required
- **FTP_DB_NAME**: db name, required
- **FTP_DB_USER**: db user, required
- **FTP_DB_PASS**: db password, required
- **FTP_ROOT**: /path/to/ftp/root, optional, defaults to /data/ftp_root
- **LOGS**: /path/to/log/dir, optional, defaults to /var/log/proftpd
- **SALT**: /path/to/salt/dir, optional, defaults to `./salt`
- **MOD_SSL**: ON/OFF, activate/deactivate module mod_tls, optional, defaults to OFF
- **SSL_CERTS**: /path/to/ssl/certs/dir, optional, defaults to `./ssl`
- **MOD_EXEC**: ON/OFF, activate/deactivate module mod_exec, optional, defaults to OFF
- **MOD_EXEC_CONF**: /path/to/mod/exec/dir, optional, defaults to `./exec`

* Build and run the container as follows:
```sh
docker-compose build
docker-compose up -d
```

### Configuring postgreSQL connection
The `FTP_DB_HOST`, `FTP_DB_NAME`, `FTP_DB_USER` and `FTP_DB_PASS` env vars should be provided to the container to configure proftpd's connection with the postgreSQL instance.

### User's passwords
Passwords are stored in the db as salted SHA256/512 digests, in hex64 encoding.

A random crypto string, known as **salt**, is used to mitigate dictionnary attacks and should be provided to the ftp server using the `SALT` env var.

The `SALT` env var let you define the directory where the `.salt` file is stored on the docker's host. Otherwise proftp will look in the `./salt` directory alongside the Dockerfile.

To generate an encrypted password use the following command:
```sh
{ echo -n myPassword; echo -n $(cat .salt); } | openssl dgst -binary -sha256 | openssl enc -base64 -A
```

where `.salt` is a file containing the **salt**.

### Server address masquerading
The server can be instructed to send back to the client a specified IP address, or hostname. This is useful when dealing with NAT gateways, or boad balancers where passive mode is required.

The env var `MASQ_ADDR` can be set to either a given IP address or hostame, or to the value `AWS` in which case the server the server public ip will be automatically retrieved (if available) from AWS EC2 instance's metadata to set the env var.

### Configuring ftp root directory
The ftp root (home for all user's directories) can be configured using the `FTP_ROOT` env variable. Otherwise it default to the directory `/data/ftp_root` of the docker's host.

### Configuring proftpd logs directory
The ftp root (home for all user's directories) can be configured using the `LOGS` env variable. Otherwise it default to the directory `/var/log/proftpd` of the docker's host.

### Module mod_tls
When enabling the module with env var MOD_EXEC=ON, a SSL certificate `proftpd.cert.pem` and it's key file `proftpd.key.pem` should be provided.

These file should be stored in a directory accessible by the docker image, whose path is to be provided as the `SSL_CERTS` env var.

### Module mod_exec
When enabling the module with env var MOD_EXEC=ON, a `exec.conf` file containing the module configuration should be provided, as per the [module's documentation](http://www.proftpd.org/docs/contrib/mod_exec.html).

This file should be stored in a directory accessible by the docker image, whose path is to be provided as the `MOD_EXEC_CONF` env var.


## Running with docker

Following the previous sections, a number a env vars and volumes needs to be specified right to the cli when running the server with docker:

- **Env vars**:
  - `FTP_DB_HOST`
  - `FTP_DB_NAME`
  - `FTP_DB_USER`
  - `FTP_DB_PASS`
  - `MASQ_ADDR`
  - `MOD_SSL`
  - `MOD_EXEC`
- **Volumes**:
  - **/srv/ftp** (_ftp root containing users' homes_)
  - **/var/log/proftpd** (_server's logs_)
  - **/etc/proftpd/salt** (_dir containing `.salt` file_)
  - **/etc/proftpd/ssl** (_dir containing server's certificates_)
  - **/etc/proftpd/exec** (_dir containing server's mod_exec conf and scripts_)

The following `docker run` example assumes bound volumes, but the anykind of docker volume config can be used.

* Build image:
```sh
docker build -t proftpd .
```

* Start container and provide the necessary env vars and volume information:
```sh
docker run --name proftpd --net=host \
  -e FTP_DB_HOST=mydb.com -e FTP_DB_NAME=db_name -e FTP_DB_USER=db_user -e FTP_DB_PASS=db_password \
  -e MASQ_ADDR:AWS \
  -v /data/ftp_root:/srv/ftp \
  -v /var/log/proftpd:/var/log/proftpd \
  -v $(pwd)/salt:/etc/proftpd/salt \
  -e MOD_SSL=ON \
  -v $(pwd)/ssl:/etc/proftpd/ssl \
  -e MOD_EXEC=ON \
  -v $(pwd)/exec:/etc/proftpd/exec \
	-d proftpd
```

_**Note**_: a Makefile is also provided in the repository to help testing the `docker run` syntax. The Makefile contains a special function `make env_run` leveraging the exact same `.env` file expected by **docker-compose**. Just make sure the `.env` file is located right next to the `Makefile` to make it work.

## Testing with curl

* Listing files:
```sh
curl -v --ssl --insecure --disable-epsv ftp://my-ftp-server.com:21 -u user:pwd
```
* Uploading files:
```sh
curl -v -T </path/to/file> --ssl --insecure --disable-epsv ftp://my-ftp-server.com:21 -u user:pwd
```
