# Spectrum Wrangler Docker

A Dockerized version of Marc DaCosta's awesome [Spectrum Wrangler](https://github.com/marcdacosta/spectrum-wrangler) repository, a utility script to import the FCC license database into a PostgreSQL database with GIS and geo indexing support.

This repo creates docker images, one for the PostgreSQL + PostGIS database and another other for the original python import script for Spectrum Wrangler. This allows for a contained environment, a custom location for the 40GB+ Postgres database, and easy automated install on remote servers.

## Installing

Before you install this software, first make sure [docker](https://www.docker.com/get-started) and [docker-compose](https://docs.docker.com/compose/install/) are installed on the host machine.

```bash
# clone the repo
git clone https://github.com/spectrum-wrangler-docker
cd spectrum-wrangler-docker

# download the dockerized brannondorsey/spectrum-wrangler fork
git submodule init
git submodule update

# build and launch the postgis/Dockerfile image. This creates a postgis container
# with it's database stored in postgis/data_directory. Note: you can't be
# running a PostgreSQL server on the default port of the host machine as ports
# will conflict.
docker-compose up postgis

# open a new terminal tab, and manually run the spectrum-wrangler docker service
# to better monitor it's logs. This service runs spectrum-wrangler/load.py
# which downloads a ~11GB CSV from the FCC's website, dumps it into the postgres
# database and creates postgis indexes on lat/long for quick spatial queries.
docker-compose up spectrum-wrangler

# optionally launch pgpadmin4 (PostgreSQL Admin tool) on http://localhost:5050.
# This can be useful if you want to make queries against the database. Simply
# create a new database connection to host: "postgis" using user and pass "fcc2"
docker-compose up pgadmin4
```

## Waiting...

Provided you don't see any errors in your console, the download should be working. This can take ~1-2hrs+ depending on your network connection and your computer's resources. Provided you don't see an error output from the `docker-compose up spectrum-wrangler` process, you should be good to go.


## Querying

The `postgis` docker service exposes the postgres service on port of the host machine. You can interact with it using whatever PostgreSQL admin tool you like, or using the `psql` shell in the docker container itself:

```bash
docker-compose exec postgis psql -U fcc2
...
psql (10.5 (Debian 10.5-1.pgdg90+1))
Type "help" for help.

fcc2=#
```

The `/tmp/fcclicense` folder in the `postgis` container is a shared volume with your host machine, so anything you save in there (like .csv files exported from postgres) will appear in the `spectrum-wrangler/fcclicense/` folder.

Here are some example queries now that you have everything setup:

```sql
-- test that the PostGIS extension has been installed and configured properly
SELECT round(dms2dd('43° 0''50.60"S'),9) as latitude, round(dms2dd('147°12''18.20"E'),9) as longitude;

-- select licenses within 1km of Trump Tower, NYC
SELECT * FROM fcclicenses WHERE ST_DWithin(
    geom::geography, 
    ST_GeogFromText('POINT(-74.0076834 40.7253319)'), 
    1000, 
    false) 
) 
```

See [`spectrum-wrangler/example-queries.sql` for several additional query examples.

## Going Further

Be default, the `fcclicense` table has a geo index on lat & long. I've found it useful to add a few additional indexes to dramatically speed up query times. You can add several additional indexes to the database like so:

```bash
# copy additional-indexes.sql to spectrum-wrangler/fcclicense/ so that it will
# appear in /tmp/fcclicense in the container's filesystem.
# Note: this may require sudo depending on how docker has created the volume
cp spectrum-wrangler/additional-indexes.sql spectrum-wrangler/fcclicense/

# add some additional indexes, this may take a while...
docker-compose exec postgis psql -U fcc2 -f /tmp/fcclicense/additional-indexes.sql fcc2 
```
