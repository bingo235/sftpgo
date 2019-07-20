# SFTPGo

Full featured and highly configurable SFTP server software

## Features

- Each account is chrooted to his Home Dir
- SFTP accounts are virtual accounts stored in a "data provider" 
- SQLite, MySQL and PostgreSQL data providers are supported. The `Provider` interface could be extented to support non SQL backends too
- Public key and password authentication
- Quota support: accounts can have individual quota expressed as max number of files and max total size
- Bandwidth throttling is supported, with distinct settings for upload and download
- Per user maximum concurrent sessions
- Per user permissions: list directories content, upload, download, delete, rename, create directories, create symlinks can be enabled or disabled
- Per user files/folders ownership: you can map all the users to the system account that runs SFTPGo (all platforms are supported) or you can run SFTPGo as root user and map each user or group of users to a different system account (*NIX only) 
- REST API for users and quota management and real time reports for the active connections with possibility of forcibly closing a connection  
- Log files are accurate and they are saved in the easily parsable JSON format
- Automatically terminating idle connections

## Platforms

SFTPGo is developed and tested on Linux, regularly the test cases are executed and pass on macOS and Windows.
Other UNIX variants such as *BSD should work too.

## Requirements

- Go 1.12 or higher
- A suitable SQL server to use as data provider: MySQL (4.1+) or SQLite 3.x or PostreSQL (9+)

## Installation

Simple install the package to your [$GOPATH](https://github.com/golang/go/wiki/GOPATH "GOPATH") with the [go tool](https://golang.org/cmd/go/ "go command") from shell:

```bash
$ go get -u github.com/drakkan/sftpgo
```

Make sure [Git is installed](https://git-scm.com/downloads) on your machine and in your system's `PATH`.

A systemd sample [service](https://github.com/drakkan/sftpgo/tree/master/init/sftpgo.service "systemd service") can be found inside the source tree.

Alternately you can use distro packages:

- Arch Linux PKGBUILD is available on [AUR](https://aur.archlinux.org/packages/sftpgo-git/ "SFTPGo")

## Configuration

The `sftpgo` executable supports the following command line flags:

- `-config-dir` string. Location of the config dir. This directory should contain the `sftpgo.conf` configuration file, the private key for the SFTP server (`id_rsa` file) and the SQLite database if you use SQLite as data provider. The server private key will be autogenerated if the user that executes SFTPGo has write access to the config-dir. The default value is "."
- `-log-file-path` string. Location for the log file, default "sftpgo.log"

Before starting `sftpgo` a dataprovider must be configured.

Sample SQL scripts to create the required database structure can be found insite the source tree [sql](https://github.com/drakkan/sftpgo/tree/master/sql "sql") directory.

The `sftpgo.conf` configuration file contains the following sections:

- **"sftpd"**, the configuration for the SFTP server
    - `bind_port`, integer the port used for serving SFTP requests. Default: 2022
    - `bind_address`, string. Leave blank to listen on all available network interfaces. Default: ""
    - `idle_timeout`, integer. Time in minutes after which an idle client will be disconnected. Default: 15
    - `umask`, string. Umask for the new files and directories. This setting has no effect on Windows. Default: "0022"
- **"data_provider"**, the configuration for the data provider
    - `driver`, string. Supported drivers are `sqlite`, `mysql`, `postgresql`
    - `name`, string. Database name
    - `host`, string. Database host. Leave empty for driver `sqlite`
    - `port`, integer. Database port. Leave empty for driver `sqlite`
    - `username`, string. Database user. Leave empty for driver `sqlite`
    - `password`, string. Database password. Leave empty for driver `sqlite`
    - `sslmode`, integer. Used for drivers `mysql` and `postgresql`. 0 disable SSL/TLS connections, 1 require ssl, 2 set ssl mode to `verify-ca` for driver `postgresql` and `skip-verify` for driver `mysql`, 3 set ssl mode to `verify-full` for driver `postgresql` and `preferred` for driver `mysql`
    - `connectionstring`, string. Provide a custom database connection string. If not empty this connection string will be used instead of build one using the previous parameters
    - `users_table`, string. Database table for SFTP users
    - `manage_users`, integer. Set to 0 to disable users management, 1 to enable
    - `track_quota`, integer. Set to 0 to disable quota tracking, 1 to update the used quota each time a user upload or delete a file
- **"httpd"**, the configuration for the HTTP server used to serve REST API
    - `bind_port`, integer the port used for serving HTTP requests. Default: 8080
    - `bind_address`, string. Leave blank to listen on all available network interfaces. Default: "127.0.0.1"

Here is a full example showing the default config:

```{
   "sftpd":{
       "bind_port":2022,
       "bind_address": "",
       "idle_timeout": 15,
       "umask": "0022"
   },
   "data_provider": {
       "driver": "sqlite",
       "name": "sftpgo.db",
       "host": "",
       "port": 5432,
       "username": "",
       "password": "",
       "sslmode": 0,
       "connection_string": "",
       "users_table": "users",
       "manage_users": 1,
       "track_quota": 1
   },
   "httpd":{
       "bind_port":8080,
       "bind_address": "127.0.0.1"
   }
}
```

## Account's configuration properties

For each account the following properties can be configured:

- `username` 
- `password` used for password authentication. The password with be stored using argon2id hashing algo
- `public_key` used for public key authentication. At least one between password and public key is mandatory
- `home_dir` The user cannot upload or download files outside this directory. Must be an absolute path
- `uid`, `gid`. If sftpgo runs as root then the created files and directories will be assigned to this system uid/gid. Ignored on windows and if sftpgo runs as non root user: in this case files and directories for all SFTP users will be owned by the system user that runs sftpgo
- `max_sessions` maximum concurrent sessions. 0 means unlimited
- `quota_size` maximum size allowed. 0 means unlimited
- `quota_files` maximum number of files allowed. 0 means unlimited
- `permissions` the following permissions are supported:
    - `*` all permission are granted
    - `list` list items is allowed
    - `download` download files is allowed
    - `upload` upload files is allowed
    - `delete` delete files or directories is allowed
    - `rename` rename files or directories is allowed
    - `create_dirs` create directories is allowed
    - `create_symlinks` create links is allowed
- `upload_bandwidth` maximum upload bandwidth as KB/s, 0 means unlimited
- `download_bandwidth` maximum download bandwidth as KB/s, 0 means unlimited

These properties are stored inside the data provider. If you want to use your existing accounts you can create a database view. Since a view is read only you have to disable user management and quota tracking so sftpgo will never try to write to the view.

## REST API

SFTPGo exposes REST API to manage users and quota and to get real time reports for the active connections with possibility of forcibly closing a connection.

If quota tracking is enabled in `sftpgo.conf` configuration file then the used size and number of files are updated each time a file is added/removed. If files are added/removed not using SFTP you can rescan the user home dir and update the used quota using the REST API.

REST API is designed to run on localhost or on a trusted network, if you need https or authentication you can setup a reverse proxy using an HTTP Server such as Apache or NGNIX.

The OpenAPI 3 schema for the exposed API can be found inside the source tree: [openapi.yaml](https://github.com/drakkan/sftpgo/tree/master/api/schema/openapi.yaml "OpenAPI 3 specs"). 

## Logs

Inside the log file each line is a JSON struct, each struct has a `sender` fields that identify the log type.

The logs can be divided into the following categories:

- **"app logs"**, internal logs used to debug `sftpgo`:
    - `sender` string. This is generally the package name that emits the log
    - `time` string. Date/time with millisecond precision
    - `level` string
    - `message` string
- **"transfer logs"**, SFTP transfer logs:
    - `sender` string. `SFTPUpload` or `SFTPDownload`
    - `time` string. Date/time with millisecond precision
    - `level` string
    - `elapsed_ms`, int64. Elapsed time, as milliseconds, for the upload/download
    - `size_bytes`, int64. Size, as bytes, of the download/upload
    - `username`, string
    - `file_path` string
    - `connection_id` string. Unique SFTP connection identifier
- **"command logs"**, SFTP command logs:
    - `sender` string. `SFTPRename`, `SFTPRmdir`, `SFTPMkdir`, `SFTPSymlink`, `SFTPRemove`
    - `level` string
    - `username`, string
    - `file_path` string
    - `target_path` string
    - `connection_id` string. Unique SFTP connection identifier
- **"http logs"**, REST API logs:    
    - `sender` string. `httpd`
    - `level` string
    - `remote_addr` string. IP and port of the remote client
    - `proto` string, for example `HTTP/1.1`
    - `method` string. HTTP method (`GET`, `POST`, `PUT`, `DELETE` etc.)
    - `user_agent` string
    - `uri` string. Full uri
    - `resp_status` integer. HTTP response status code
    - `resp_size` integer. Size in bytes of the HTTP response
    - `elapsed_ms` int64. Elapsed time, as milliseconds, to complete the request
    - `request_id` string. Unique request identifier
    
## Acknowledgements

- [pkg/sftp](https://github.com/pkg/sftp)
- [go-chi](https://github.com/go-chi/chi)
- [zerolog](https://github.com/rs/zerolog)
- [lumberjack](https://gopkg.in/natefinch/lumberjack.v2)
- [argon2id](https://github.com/alexedwards/argon2id)
- [go-sqlite3](https://github.com/mattn/go-sqlite3)
- [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)
- [lib/pq](https://github.com/lib/pq)

Some code was initially taken from [Pterodactyl sftp server](https://github.com/pterodactyl/sftp-server)

## License

GNU GPLv3