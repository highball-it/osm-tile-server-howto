# Tile Server

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Tile Server](#tile-server)
	- [TODOs](#todos)
	- [Installation and configuration](#installation-and-configuration)
		- [Optional: Install VirtualBox Guest Additions](#optional-install-virtualbox-guest-additions)
		- [Install Postgres und PostGIS](#install-postgres-und-postgis)
		- [Install osm2pgsql](#install-osm2pgsql)
		- [Download and import planet files](#download-and-import-planet-files)
		- [Install apache2, apache2-dev and autoconf for mod_tile](#install-apache2-apache2-dev-and-autoconf-for-modtile)
		- [Install mapnik](#install-mapnik)
		- [Install mod_tile](#install-modtile)
		- [Install styles for mapnik](#install-styles-for-mapnik)
		- [Configure renderd, add systemd entry](#configure-renderd-add-systemd-entry)
	- [Setup test website](#setup-test-website)
	- [Nominatim](#nominatim)

<!-- /TOC -->

## TODOs
* Update planet date in database
* Add whole planet (?)
* Add Nominatim server

## Installation and configuration

### Optional: Install VirtualBox Guest Additions
    sudo apt-get install build-essential linux-headers-$(uname -r)

    sudo mount -t iso9660 /dev/sr0 /media/cdrom
    cd /media/cdrom
    sudo ./VBoxLinuxAdditions.run

### Install Postgres und PostGIS
Install packages:

    sudo apt-get install postgresql postgresql-contrib postgis

Create database user and database:

    sudo -u postgres createuser gisuser
    sudo -u postgres createdb --encoding=UTF8 --owner=gisuser gis

Enable PostGIS extension for `gis` database:

    sudo -u postgres psql --username=postgres --dbname=gis -c "CREATE EXTENSION postgis;"
    sudo -u postgres psql --username=postgres --dbname=gis -c "CREATE EXTENSION postgis_topology;"

See also: [OSM Wiki PostGIS Installation](https://wiki.openstreetmap.org/wiki/PostGIS/Installation)

### Install osm2pgsql
Install packages:

    sudo apt-get install osm2pgsql

See also: [OSM Wiki Osm2pgsql](https://wiki.openstreetmap.org/wiki/Osm2pgsql#For_Debian_or_Ubuntu)

### Download and import planet files
Initially we have to fill the database with the planet files. Later we can use kind of diff files.

If you want to import only a few parts you have to merge them into one file to avoid errors.

    sudo apt-get install osmctools

    sudo su - postgres

    wget http://download.geofabrik.de/europe/germany/bremen-latest.osm.pbf
    wget http://download.geofabrik.de/europe/germany/hamburg-latest.osm.pbf
    wget http://download.geofabrik.de/europe/germany/schleswig-holstein-latest.osm.pbf

    osmconvert bremen-latest.osm.pbf | osmconvert - hamburg-latest.osm.pbf | osmconvert - schleswig-holstein-latest.osm.pbf -o=norden.osm.pbf

    osm2pgsql --slim -C 1500 --number-processes 4 norden.osm.pbf

In `osm2pgsql --slim -C <cache size> --number-processes 4 planet-latest.osm.pbf` the `<cache size>` is about 75% of memory in MiB, to a maximum of about 30000. Additional RAM will not be used. The database to use can be specified by `-d <db name>`, the default is `gis`.

See also: [Osm2pgsql @ GitHub](https://github.com/openstreetmap/osm2pgsql), [OSM Wiki Osm2pgsql](https://wiki.openstreetmap.org/wiki/Osm2pgsql), `man osm2pgsql`, [OSM Wiki osmconvert](https://wiki.openstreetmap.org/wiki/Osmconvert)

### Install apache2, apache2-dev and autoconf for mod_tile
Install packages:

    sudo apt-get install apache2 apache2-dev autoconf

### Install mapnik
Install packages:

    sudo apt-get install libmapnik3.0 libmapnik-dev mapnik-utils python-mapnik

### Install mod_tile
Clone, compile and install:

    git clone https://github.com/openstreetmap/mod_tile.git
    cd mod_tile/
    ./autogen.sh
    ./configure
    make
    sudo make install
    sudo make install-mod_tile

Add file for module to apache and enable mod_tile:

    echo "LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so" | sudo tee /etc/apache2/mods-available/mod_tile.load
    sudo a2enmod mod_tile
    sudo systemctl restart apache2.service

### Install styles for mapnik

    sudo apt-get install unzip

    git clone https://github.com/openstreetmap/mapnik-stylesheets.git
    cd mapnik-stylesheets/
    sudo ./get-coastlines.sh /usr/local/share

    cd inc/
    sudo cp datasource-settings.xml.inc.template datasource-settings.xml.inc
    sudo cp fontset-settings.xml.inc.template fontset-settings.xml.inc
    sudo cp settings.xml.inc.template settings.xml.inc

Modify the files as described:
datasource-settings.xml.inc

    <!--
    Settings for your postgres setup.
    Note: feel free to leave password, host, port, or use blank
    -->
    <Parameter name="type">postgis</Parameter>
    <!-- <Parameter name="password">%(password)s</Parameter> -->
    <!-- <Parameter name="host">%(host)s</Parameter> -->
    <!-- <Parameter name="port">%(port)s</Parameter> -->
    <!-- <Parameter name="user">%(user)s</Parameter> -->
    <Parameter name="dbname">gis</Parameter>
    <!-- this should be 'false' if you are manually providing the 'extent' -->
    <Parameter name="estimate_extent">false</Parameter>
    <!-- manually provided extent in epsg 900913 for whole globe -->
    <!-- providing this speeds up Mapnik database queries -->
    <Parameter name="extent">-20037508,-19929239,20037508,19929239</Parameter>

settings.xml.inc
Replace the following lines:

    <!ENTITY symbols "%(symbols)s">
    <!ENTITY osm2pgsql_projection "&srs%(epsg)s;">
    <!ENTITY dwithin_node_way "&dwithin_%(epsg)s;">
    <!ENTITY world_boundaries "%(world_boundaries)s">
    <!ENTITY prefix "%(prefix)s">

with:

    <!ENTITY symbols "symbols">
    <!ENTITY osm2pgsql_projection "&srs900913;">
    <!ENTITY dwithin_node_way "&dwithin_900913;">
    <!ENTITY world_boundaries "/usr/local/share/world_boundaries">
    <!ENTITY prefix "planet_osm">



_Notice: The script is not working well with IPv6, so maybe add `-4` to the wget call in the script._  
_Notice: Maybe we need to delete the temp archives from `/usr/local/share`_  
_Notice: Find out where to put the mapnik-stylesheets folder!_

### Configure renderd, add systemd entry

    sudo cp /usr/local/etc/renderd.conf /etc/

    sudo mkdir -p /var/lib/mod_tile
    sudo touch /var/lib/mod_tile/planet-import-complete
    sudo chown -R www-data /var/lib/mod_tile

/etc/renderd.conf

    [renderd]
    ;socketname=/var/run/renderd/renderd.sock
    num_threads=4
    tile_dir=/var/lib/mod_tile
    stats_file=/var/run/renderd/renderd.stats

    [mapnik]
    plugins_dir=/usr/lib/mapnik/3.0/input
    font_dir=/usr/share/fonts/truetype
    font_dir_recurse=1

    [default]
    URI=/osm_tiles/
    TILEDIR=/var/lib/mod_tile
    XML=/srv/mapnik-stylesheets/osm.xml
    HOST=localhost
    TILESIZE=256
    ;MINZOOM=0
    MAXZOOM=20
    ;TYPE=png image/png
    ;DESCRIPTION=This is a description of the tile layer used in the tile json request
    ;ATTRIBUTION=&copy;<a href=\"http://www.openstreetmap.org/\">OpenStreetMap</a> and <a href=\"http://wiki.openstreetmap.org/wiki/Contributors\">contributors</a>, <a href=\"http://opendatacommons.org/licenses/odbl/\">ODbL</a>
    ;SERVER_ALIAS=http://localhost/
    ;CORS=http://www.openstreetmap.org
    ;ASPECTX=1
    ;ASPECTY=1
    ;SCALE=1.0

Configure postgres auth:

In /etc/postgresql/9.5/main/pg_hba.conf modify:

    local   all             all                                     ident map=gisuser

In /etc/postgresql/9.5/main/pg\_ident.conf

    # MAPNAME       SYSTEM-USERNAME         PG-USERNAME
    gisuser         www-data                gisuser

Restart postgresql

    sudo systemctl restart postgresql

Add systemd entry:

    sudo tee /etc/systemd/system/renderd.service <<EOF
    [Unit]
    Description=OSM render deamon
    [Service]
    Type=fork
    User=www-data
    Group=www-data
    RuntimeDirectory=renderd
    ExecStart=/usr/local/bin/renderd -f
    [Install]
    WantedBy=multi-user.target
    EOF

    sudo systemctl enable renderd

_Notice: The option `-f` runs `renderd` in foreground mode. Disable this if you no longer need debug output in syslog._

## Setup test website

    sudo cp mod_tile/mod_tile.conf /etc/apache2/sites-available/

/etc/apache2/sites-available/mod\_tile.conf (minimal)

    <VirtualHost 127.0.0.1:80>
        ServerName localhost
        DocumentRoot /var/www/html

        ModTileTileDir /var/lib/mod_tile
        LoadTileConfigFile /etc/renderd.conf

        ModTileEnableStats On

        ModTileBulkMode Off
        ModTileRequestTimeout 3
        ModTileMissingRequestTimeout 10
        ModTileMaxLoadOld 16
        ModTileMaxLoadMissing 50
        ModTileVeryOldThreshold 31536000000000
        ModTileRenderdSocketName /var/run/renderd/renderd.sock

        ModTileCacheDurationMax 604800
        ModTileCacheDurationDirty 900
        ModTileCacheDurationMinimum 10800
        ModTileCacheDurationMediumZoom 13 86400
        ModTileCacheDurationLowZoom 9 518400
        ModTileCacheLastModifiedFactor 0.20

        ModTileEnableTileThrottling Off
        ModTileEnableTileThrottlingXForward 0
        ModTileThrottlingTiles 10000 1
        ModTileThrottlingRenders 128 0.2

        LogLevel debug
    </VirtualHost>

_Notice: For further information on mod\_tile configuration see_ [mod\_tile.conf @ GitHub](https://github.com/openstreetmap/mod_tile/blob/master/mod_tile.conf)

index.html

    <!DOCTYPE html>
    <html>
    <head>
        <title>Quick Start - Leaflet</title>
        <meta charset="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <link rel="shortcut icon" type="image/x-icon" href="docs/images/favicon.ico" />
        <link rel="stylesheet" href="https://unpkg.com/leaflet@1.0.3/dist/leaflet.css" />
        <script src="https://unpkg.com/leaflet@1.0.3/dist/leaflet.js"></script>
    		<style>
    				html, body, #mapid {
      					width: 100%; height: 100%;
      					margin:0; padding: 0;
    				}
    		</style>        
    </head>
    <body>
    		<div id="mapid"></div>
    		<script>
        		var mymap = L.map('mapid').setView([53.86840, 10.68586], 13);
            L.tileLayer('http://localhost/osm_tiles/{z}/{x}/{y}.png', {
                    maxZoom: 20,
                    attribution: 'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, ' +
                            '<a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, ' +
                            'Imagery © 2017 Localhost'
            }).addTo(mymap);
    		</script>
    </body>
    </html>

## Nominatim
See: [OSM Wiki Nominatim](https://wiki.openstreetmap.org/wiki/Nominatim)  
See: [Nominatim @ GitHub](https://github.com/twain47/Nominatim)
