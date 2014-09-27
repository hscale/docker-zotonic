docker-zotonic
==============

Docker packaging for Zotonic. There are adapted versions in subfolders for
different versions of Zotonic.

Numbered version folders contain packaging using ZIP release versions of
Zotonic repackaged as tarballs. This is because a tarball works better as
a source format for Docker. The master release version is a running build
using the master branch from git to build Zotonic during Docker image build.

Environment variables
---------------------

On container startup both global default configuration file and each site
specific configuration is run through a conversion script. This script will
check for the existence of environment variables and if the environment
variable is set, it will replace the value of the variable in the file with
the one from the environment variable.

For the global defaults file all of the variables listed below are considered.
For the site specific configuration only database connection parameters
are considered. This is so that a database provided by container linking can
be connected to. The connection details may not be known before the container
is running with a link.

Use the usual Docker mechanisms to set variables that you need to override
defaults from configuration files. Please note that modifying site specific
configuration files provided from a data volume or similar is likely a better
option than setting environment variables.

Easiest option for setting parameters is using an ```env``` file and using the
```--env-file``` option for ```docker run```.

### Global default admin password

`ADMINPASSWORD=supersecret`

### Database variables 

Host and port are likely to be set by linking but can be overridden by variable.
You can set protocol to anything, like 'tcp', but the value is not used.
It is important that the value is in the 3 colon format however.

```
DB_PORT=<protocol>://<host_or_ip>:<port>

DB_USERPASS=<database_user>:<database_password>
or
DBPASSWORD=<database_password>
DBUSER=<database_user>

DB_DATABASE=<database_name>
DB_SCHEMA=<database_schema>
```

Linking
-------

The most likely option for connecting to a database is linking to a container
running PostgreSQL. Please see [Postgres at Docker Hub][] for details on running
a postgres instance in a container.

Once the PostgreSQL container is running, and populated with any site data if
necessary, you may link to it with the ```--link postgres:db``` option for
```docker run```. **NOTE**: The name of the container running postgres is not
relevant but it is very important that the alias used is **db**. The alias is
used for the environment variables inside the container and those are parsed
to override Zotonic site configuration.

Data Volumes
------------

A good way to add site data is to use [data volume containers][]. As an example
let's assume there is a container called ```sites-data``` running which has
been setup with [data volumes][] for sites and other necessary data. The
Zotonic container can then be started using the ```--volumes-from sites-data```
option for ```docker run```.

Also note that a data volume can also be just a single file as seen in the case
of the global ```config.in``` file below. You may provide a modified version of the
config.in file for example to set SMTP settings and on container start the
database settings are set by the script as described in [Environment variables][]

However data volumes are set up Zotonic should have access to the following
locations[^containerlocations] as persistent [data volumes][]:

### Zotonic 0.11 and later ###

```
/home/zotonic/.zotonic/0.11/
/srv/zotonic/priv/log
/srv/zotonic/user/sites
```

### Zotonic 0.10.1 and prior ###

```
/srv/zotonic/priv/config.in
/srv/zotonic/priv/log
/srv/zotonic/priv/sites
```

[^containerlocations]: The locations shown are the path inside the Zotonic
container. The host path may be something completely different.

Examples
--------

### Start a database container with data from host ###

```bash
$sudo docker run -d --name db -v /mnt/db/data:/var/lib/postgresql/data postgres
```

### Run a data volume container for persistent data ###
```bash
$sudo docker run -v /mnt/sites/:/srv/zotonic/priv/sites/ \
  -v /mnt/zotonic/priv/config.in:/srv/zotonic/priv/config.in \
  -v /mnt/zotonic/priv/log/:/srv/zotonic/priv/log \
  --name zotonic-data busybox true
```

### Start a Zotonic container

Database is linked to from *db* above and data volumes from *zotonic-data*.
Also note the use of a file containing environment variables.

```bash
$sudo docker run -p 80:8000 -p 443:8443 --volumes-from zotonic-data \
  --link db:db --env-file zotonic.env --rm -t zotonic
```

```
$cat zotonic.env 
ADMINPASSWORD=supersecret_default_pass
```

By default the container will expose port 8000. In this case the site has been
setup to use SSL and the ports are exposed accordingly to the host ports
80 and 443.

### Start Zotonic container to shell bypassing the startup script

```bash
$sudo docker run -p 80:8000 -p 443:8443 --volumes-from zotonic-data --rm -ti \
  --entrypoint /bin/bash zotonic
```

[Postgres at Docker Hub]: https://registry.hub.docker.com/_/postgres/
[data volumes]: http://docs.docker.com/userguide/dockervolumes/#data-volumes
[data volume containers]: http://docs.docker.com/userguide/dockervolumes/#creating-and-mounting-a-data-volume-container
