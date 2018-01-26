# osm-server

This provides an open street map tile server inside Docker containers.

[Installation](doc/Install.md)

## Updating the tile server
This refers to a map update, which requires a whole new database import of the latest data.
1. Make sure the server is running.
2. Set environment variables and run the import command as in the [Populate the database](#populate-the-database) section. Note this will clear the database before repopulating it. If you want to keep the old database then you'll need to backup the `osmserver_osm_postgres_database` volume. [Here's an example of how to do that](https://loomchild.net/2017/03/26/backup-restore-docker-named-volumes/).
3. Clear pre-rendered tiles as in the [clear pre-rendered or cached tiles](#clear-pre-rendered-or-cached-tiles) section.
4. Restart the tile server as in the [Restart the tile server](#restart-the-tile-server) section.

## Customising data import
You may wish to customise the parameters which are passed to `osm2pgsql` which is the program which imports the data into the database. Changing these can vary the speed of the import quite significantly. In particular, giving it more RAM helps. You can edit the `docker-compose-importer.yml` file to change the `osm2pgsql` command. The `--cache` argument is how many MB of RAM can be used for the import. The `--cache-strategy` is about how memory is allocated (in one block or sparsely). More information on the import can be found [here](https://wiki.openstreetmap.org/wiki/Osm2pgsql#Optimization) and [here](http://www.volkerschatz.com/net/osm/osm2pgsql-usage.html).

## Tile pre-rendering
You can force the rendering of all or a subset of map tiles. This means that when they are requested it will be faster, because the png will only need to be fetched from disk rather than having to be generated from the database as well. The `render_list_geo` perl script does this. See the README for [this repository](https://github.com/alx77/render_list_geo.pl) for more information on how to use the command..

In the following command, you will need to replace `osmserver_tile-server_1` with the name of the running tile server docker container. You should be able to get the name of it with `docker ps | grep tile-server`.

This example renders zoom levels 0 to 15 for the UK. (z is zoom, x is longitude, y is latitude). See also [the original readme](https://github.com/alx77/render_list_geo.pl).

`docker exec -it osmserver_tile-server_1 render_list_geo -z 0 -Z 15 -x -8 -X 1.8 -y 49.8 -Y 60.9`

## Clear pre-rendered or cached tiles
In the following command, you will need to replace `osmserver_tile-server_1` with the name of the running tile server docker container. You should be able to get the name of it with `docker ps | grep tile-server`. This will clear all cached tiles:

`docker exec -it osmserver_tile-server_1 rm -r /var/lib/mod_tile/default`

## Destroy the tile server
To clean everything up, remove the containers and the volume containing the postgres database with
```
./destroy_tile_server    # Linux
destroy_tile_server.bat  # Linux type shell on Windows
```

### Hardware requirements
**Memory:** The more memory you can give the process, the better. Typically `--cache 4000` is enough for it not to be too slow, but `--cache 64000` on a beefy computer would be faster. The `--cache-strategy` flag can also make a difference.

**Disk:** To import the USA you'll need about 250GB of disk space.

**CPU:** For rendering tiles quickly, more cores and faster speeds will lead to tiles loading faster for the user.

## How it works
There are three components to this system
### 1. Postgres database
This stores the OSM map data in the form of vertices, lines, etc. with associated metadata. The docker image used for this is a basic Postgres database with some extensions specific to OSM added.
### 2. Web tile server/renderer
When a tile is requested at a certain zoom level and location, this container queries the database for the relevant data and then renders it into a png image. This image is then cached inside the container, so expect this container to grow in size over time.
### 3. Data importer
This isn't part of the running system, but this container contains `osm2pgsql`, a program to connect to the postgres database and populate it with data from a `.pbf` file.

### Storage
* The database is stored in a docker volume which is mounted into the postgres container. This volume is created during the `docker-compose up` command. When `docker-compose down` destroys the containers, this volume is not destroyed but should not be reused - you can manually destroy it.
* The rendered png tiles are cached inside the web server container. As such the tile server container will grow in size over time.

### Networking
* A network is connected to by all three containers. This allows them to communicate with each other.

[Customising the map style](doc/MapStyle.md)
[Provisioning a Centos Tile Server](doc/ServerProvisioning.md)