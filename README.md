
# SOTM Asia 2022: Full Stack Web Mapping Workshop

### Tools need to be installed

1. Postgresql & PostGIS
  
2. osm2pgsql
  
3. gdal/ogr2ogr
  
4. Tippecanoe
  
5. Fontnik
  
6. Spritezero
  
7. Tileserver-GL
  
8. Maputnik
  
9. OSRM
  
10. Valhalla
  
11. Graphhopper
  
12. Pelias
  

## Add `sudo` user

```bash
adduser geodevops
```

Then Enter Password and add the user to `sudo` group

```bash
usermod -aG sudo geodevops
sudo chown -R geodevops:geodevops /opt
```

## Install Postgresql & PostGIS

#### 1) Enable PostgreSQL Package Repository

PostgreSQL 15 package is not available in the default package repository, so enable its official package repository using following commands.

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc 
```

To begin, let’s fetch the latest versions of the packages. We can achieve this using the `apt update` command as shown below:

```bash
sudo apt update
```

#### 2) Install PostgreSQL 15 Database Server and Client

The postgresql package installs the default version of the PostgreSQL database server whereas the postgresql-client package installs the client utility.

Let’s install the PostgreSQL client and server using the below apt command:

```bash
sudo apt install postgresql postgresql-client -y
```

Next, let’s verify that the PostgreSQL service is up and running:

```bash
sudo systemctl status postgresql
```

Finally, check the PostgreSQL version using the psql command line utility:

```bash
psql --version
```

#### 3) Change `postgresql.config` and `pg_hba.conf` file

open `postgresql.conf` file by running following command

```bash
sudo nano /etc/postgresql/15/main/postgresql.conf
```

change the following line of **postgresql.conf** file

```apacheconf
listen_addresses = '*'
```

Now open `pg_hba.conf` file by running following command

```bash
sudo nano /etc/postgresql/15/main/pg_hba.conf
```

Add the following line at the end of the **pg_hba.conf** file

```apacheconf
host    all         all        0.0.0.0/0           md5
```

Restart postgresql service, allow port and install `libpq-dev` dependency

```bash
sudo systemctl restart postgresql
sudo ufw allow 5432/tcp
sudo apt install libpq-dev -y
sudo systemctl status postgresql
```

#### 4) Update PostgreSQL `postgres` User Password

```bash
sudo su - postgres
psql -c "alter user postgres with password 'StrongPassword'"
```

#### 5) Create a Super User - `admin` and create a database

```
createuser admin
createdb osm -O admin
psql -l  | grep osm
psql
```

Now change the password of `admin` user and provide super user privilege

```sql
ALTER USER admin WITH password 'AdminPassword';
ALTER USER admin WITH SUPERUSER;
\q
```

#### 6) Install PostGIS and enable PostGIS to database

```bash
sudo apt install -y postgresql-15-postgis-3
sudo apt install -y postgresql-15-postgis-3-scripts #Already added with PostGIS version 3
```

Enter the database as `admin` user

```bash
psql -h localhost -p 5432 -U admin osm
```

Now create ecessary extensions

```sql
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
CREATE EXTENSION hstore;
\q
```

## Install osm2pgsql and Insert OSM data into Postgresql Database

#### 1) Donwload OSM Data and Clip a small area

Download the OSM data of Philippines

```bash
wget -O map.osm.pbf https://download.geofabrik.de/asia/philippines-latest.osm.pbf
```

To clip osm data `osmconvert` will be needed and need to be installed

```bash
sudo apt-get install osmctools
```

Now clip osm data using the following command

```bash
# Legazpi
osmconvert map.osm.pbf -b=123.70933319496403,13.135874451375173,123.76493813722841,13.160187971225412 -o=legazpi.osm.pbf
#Manila
osmconvert map.osm.pbf -b=121.00404997541462,14.546196395828636,121.04521633510136,14.568028082827874 -o=makati.osm.pbf
```

#### 2) Install osm2pgsql and insert data into database

Install osm2pgsql

```bash
sudo apt-get install osm2pgsql -y
```

We need a style/template file to store osm data. Download the file from here

```bash
curl -O -J -L "https://eu2.contabostorage.com/ecf3feb4159740a4bd937c3d68be7234:dl.locusq/default.style"
```

Store data from osm to pgsql

```bash
osm2pgsql -c -d osm -U admin -H localhost -P 5432 -W --hstore -C 4000 --number-processes 16 -S default.style makati.osm.pbf
```

## Gdal/ogr2ogr

#### 1) Install gdal

```bash
sudo add-apt-repository ppa:ubuntugis/ppa
sudo apt-get update
sudo apt-get install gdal-bin -y
ogrinfo --version
```

#### 2) Create GeoJSON Layers

```bash
### 1. landuse
ogr2ogr -progress -f GeoJSON landuse.geojson -t_srs EPSG:4326 "PG:host=localhost port=5432 dbname=osm user=admin password=AdminPassword"  -sql "select osm_id, name, landuse as type, way from planet_osm_polygon where landuse is not null"

### 2. waterbodies_high
ogr2ogr -progress -f GeoJSON waterbodies.geojson -t_srs EPSG:4326 "PG:host=localhost port=5432 dbname=osm user=admin password=AdminPassword"  -sql "select osm_id, name, water as type, way from planet_osm_polygon where water is not null"

### 3. Major Road
ogr2ogr -progress -f GeoJSON major_road.geojson -t_srs EPSG:4326 "PG:host=localhost port=5432 dbname=osm user=admin password=AdminPassword"  -sql "SELECT way, highway AS type, name
from planet_osm_line
where highway IN ('motorway', 'trunk', 'primary', 'secondary', 'tertiary')
order by z_order"

### 4. Minor Road
ogr2ogr -progress -f GeoJSON minor_road.geojson -t_srs EPSG:4326 "PG:host=localhost port=5432 dbname=osm user=admin password=AdminPassword"  -sql "SELECT way, highway AS type, name
from planet_osm_line
where highway NOT IN ('motorway', 'trunk', 'primary', 'secondary', 'tertiary')
order by z_order"

### 5. Buildings
ogr2ogr -progress -f GeoJSON buildings.geojson -t_srs EPSG:4326 "PG:host=localhost port=5432 dbname=osm user=admin password=AdminPassword"  -sql "SELECT way, building AS type, COALESCE (tags -> 'name:en',name) as en,
 (case when (tags -> 'building:levels') ~ '^[0-9]*$' then (tags -> 'building:levels')::integer
else 1 end)
 as building_height
FROM planet_osm_polygon p
WHERE building NOT IN ('0','false', 'no')
ORDER BY ST_YMin(ST_Envelope(way)) desc"
```

## Tippecanoe

#### 1) Install Tippecanoe

```bash
sudo apt-get install build-essential libsqlite3-dev zlib1g-dev -y
git clone https://github.com/mapbox/tippecanoe.git
cd tippecanoe
make -j
sudo make install
```

#### 2) Convert GeoJSON to MBTiles

```bash
### 1. Landuse MBTiles
tippecanoe -Z5 -z18 -f  -o landuse.mbtiles --drop-densest-as-needed --extend-zooms-if-still-dropping --drop-smallest-as-needed --simplification=4 --simplify-only-low-zooms landuse.geojson

### 2. Waterbodies MBTiles
tippecanoe -Z8 -z18 -f  -o waterbodies.mbtiles --drop-densest-as-needed --extend-zooms-if-still-dropping --drop-smallest-as-needed --simplification=4 --simplify-only-low-zooms waterbodies.geojson

### 3. Major Roads MBTiles
tippecanoe -Z5 -z18 -pk -pC -pg -f  -o major_road.mbtiles --drop-densest-as-needed --extend-zooms-if-still-dropping --drop-smallest-as-needed --simplification=4 --simplify-only-low-zooms major_road.geojson

### 4. Minor Roads MBTiles
tippecanoe -Z5 -z18 -pk -pC -pg -f  -o minor_road.mbtiles --drop-densest-as-needed --extend-zooms-if-still-dropping --drop-smallest-as-needed --simplification=4 --simplify-only-low-zooms minor_road.geojson

### 5. Buildings MBTiles
tippecanoe -Z13 -z18 -pk -pC -pg -f -o buildings.mbtiles --drop-densest-as-needed --extend-zooms-if-still-dropping --drop-smallest-as-needed --simplification=4 --simplify-only-low-zooms buildings.geojson
```

##### Join The Tiles

```bash
# Join MBTiles
tile-join -f -pk -pC -pg -n makati -o makati.mbtiles landuse.mbtiles waterbodies.mbtiles major_road.mbtiles minor_road.mbtiles buildings.mbtiles
```

## Fontnik (Not Done)

#### 1) Install Node & npm

```bash
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.39.0/install.sh | bash
source /home/geodevops/.bashrc
nvm --version
```

Install node 6,10 &12

```bash
nvm install 6.15.1
nvm install 10.16.2
nvm install 12.16.2
```

#### 2) Install Fontnik

```bash
nvm use 10.16.2
git clone https://github.com/mapbox/node-fontnik.git
cd node-fontnik
npm install
```

#### 3) Create a custom glyphs

```bash
cd fonts
wget -O fonts.zip  https://fonts.google.com/download\?family\=Lobster%20Two
unzip fonts.zip
cd ..
mkdir -p glyphs/lobstar
cd ..
bin/build-glyphs fonts/LobsterTwo-Regular.ttf glyphs/lobstar
```

## Spritezero (Not Done)

#### 1) Install Spritezero (node 6.15.1)

```bash
nvm use 6.15.1
npm config set user 0
npm config set unsafe-perm true
mkdir spritezero
cd spritezero
npm install -g @mapbox/spritezero-cli
mkdir sprite
```

#### 2) Create custom Icon

```bash
wget -O maki.zip https://github.com/mapbox/maki/zipball/main
unzip maki.zip

mv mapbox-maki-* maki
spritezero sprite maki/icons/
mv sprite.json sprite.png ./sprite
spritezero sprite maki/icons/ --ratio=2
mv sprite.json sprite@2x.json
mv sprite.png sprite@2x.png
mv sprite@2x.json sprite@2x.png ./sprite
```

## Tileserver-GL

#### 1) Install Tileserver-GL

```bash
sudo apt-get install -y software-properties-common protobuf-compiler pkg-config libcairo2-dev libjpeg-dev libgif-dev git libgl1-mesa-glx build-essential g++ curl

sudo apt-get install -y libegl1-mesa libgles2-mesa

git clone --depth 1 --branch v3.0.0 https://github.com/maptiler/tileserver-gl.git

cd tileserver-gl
nvm use 10.16.2
npm_config_user=root npm install -g
```

If `sqlite3` or `canvas` error appears

```bash
npm uninstall sqlite3 
npm install --save sqlite3 
npm rebuild

npm rebuild --verbose sharp
npm uninstall canvas
npm i canvas
```

#### Create Tileserver-GL `config` file

```json
{
  "options": {
    "paths": {
      "root": "",
      "fonts": "fonts",
      "styles": "styles",
      "mbtiles": "mbtiles"
    },
    "formatQuality": {
      "jpeg": 80,
      "webp": 90
    },
    "maxScaleFactor": 3,
    "maxSize": 2048,
    "pbfAlias": "pbf",
    "serveAllFonts": true,
    "serveStaticMaps": true
  },
  "styles": {
    "mymap": {
      "style": "mymap/style.json",
      "serve_rendered": false,
      "tilejson": {
        "type": "overlay",
        "bounds": [
          121.00404997541462,
          14.546196395828636,
          121.04521633510136,
          14.568028082827874
        ]
      }
    }
  },
  "data": {
    "makati": {
      "mbtiles": "makati.mbtiles"
    }
  }
}
```

##### Create Style File

```json
{
  "version": 8,
  "name": "web_map_workshop",
  "metadata": {
    "mapbox:autocomposite": true,
    "mapbox:type": "template",
    "maputnik:renderer": "mbgljs"
  },
  "center": [
    121.00404,
    14.54619
  ],
  "zoom": 11.6,
  "bearing": 0,
  "pitch": 0,
  "sources": {
    "makati": {
      "type": "vector",
      "url": "mbtiles://makati.mbtiles"
    }
  },
  "sprite": "{styleJsonFolder}/sprite",
  "glyphs": "{fontstack}/{range}.pbf",
  "layers": [
    {
      "id": "background",
      "type": "background",
      "minzoom": 5,
      "maxzoom": 24,
      "paint": {
        "background-color": "rgba(245, 245, 241, 1)"
      }
    }
  ],
  "id": "web_map_workshop",
  "owner": "SOTM"
}
```

#### 3) Run Tileserver-GL

##### Basic Preview with Switzerland

```bash
wget https://github.com/maptiler/tileserver-gl/releases/download/v1.3.0/zurich_switzerland.mbtiles
tileserver-gl --mbtiles zurich_switzerland.mbtiles
```

```bash
tileserver-gl --config config.json --port 8888
```

## Maputnik - Map Design Tool

#### 1) Build Maputnik

```bash
git clone https://github.com/maputnik/editor.git
cd editor
nano config/webpack.config.js
```

change host to 0.0.0.0

```bash
host=0.0.0.0,
#or
const HOST = process.env.HOST || "0.0.0.0";
```

add line disableHostCheck: true, after port

```bash
disableHostCheck: true,
```

install dependencies (Node 12.16.2)

```bash
npm install
```

It will take 7 minutes to install the Maputnik. start Maputnik server

```bash
npm start
```

## OSRM

#### 1) Build OSRM

```bash
mkdir -p osrm/data && $_
cp /opt/osm_data/makati.osm.pbf .
```

Install Dependencies

```bash
sudo apt install build-essential git cmake pkg-config libbz2-dev libstxxl-dev libstxxl1v5 libxml2-dev libzip-dev libboost-all-dev lua5.2 liblua5.2-dev libtbb-dev libluabind-dev libluabind0.9.1d1
```

Install OSRM-Backend

```bash
cd ..
git clone https://github.com/Project-OSRM/osrm-backend.git
mkdir -p osrm-backend/build && $_
cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_MASON=ON
cmake --build .
sudo cmake --build . --target install
```

#### 2) Prepare routing data & Run Server

```bash
osrm-extract --profile ../profiles/car.lua data/makati.osm.pbf
osrm-contract data/makati.osrm
osrm-routed data/makati.osrm --port 5000
```

Test: http://174.138.26.214:5000/route/v1/driving/121.0147874254375,14.558760176649304;121.02491749406659,14.560294482680346?steps=true

121.0147874254375,14.558760176649304

121.02491749406659,14.560294482680346

## VALHALLA

## Building Valhalla

```bash
mkdir /opt/valhalla && cd $_
mkdir /opt/valhalla/building_valhalla && cd $_
sudo apt-get update
sudo apt-get install -y git wget curl ca-certificates gnupg2
sudo apt-get install -y cmake build-essential
```

```bash
sudo apt-get install -y libsqlite3-mod-spatialite libsqlite3-dev libspatialite-dev \
     autoconf libtool pkg-config libczmq-dev libzmq5 \
     libcurl4-openssl-dev zlib1g-dev jq libgeos-dev \
     libgeos++-dev libprotobuf-dev protobuf-compiler \
     libboost-all-dev liblua5.2-dev spatialite-bin unzip

sudo ln -s /usr/lib/x86_64-linux-gnu/mod_spatialite.so /usr/lib/mod_spatialite

git clone https://github.com/kevinkreiser/prime_server.git
cd prime_server
git submodule update --init --recursive

./autogen.sh
./configure --prefix=/usr LIBS="-lpthread"

make all -j$(nproc)
make -j$(nproc) -k test
sudo make install
cd ../

git clone https://github.com/valhalla/valhalla
cd valhalla
git submodule sync
git submodule update --init --recursive

curl -o- curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[[ -s "$NVM_DIR/nvm.sh" ]] && \. "$NVM_DIR/nvm.sh"
nvm install 10.16.2 && nvm use 10.16.2
npm install --ignore-scripts --unsafe-perm=true

sudo ln -s ~/.nvm/versions/node/v10.16.2/include/node/node.h /usr/include/node.h
sudo ln -s ~/.nvm/versions/node/v10.16.2/include/node/uv.h sudo /usr/include/uv.h
sudo ln -s ~/.nvm/versions/node/v10.16.2/include/node/v8.h /usr/include/v8.h

mkdir build
sudo cmake . -Bbuild \
 -DCMAKE_C_FLAGS:STRING="${CFLAGS}" \
 -DCMAKE_CXX_FLAGS:STRING="${CXXFLAGS}" \
 -DCMAKE_EXE_LINKER_FLAGS:STRING="${LDFLAGS}" \
 -DCMAKE_INSTALL_LIBDIR=lib \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DENABLE_DATA_TOOLS=On \
 -DENABLE_PYTHON_BINDINGS=On \
 -DENABLE_NODE_BINDINGS=Off \
 -DENABLE_SERVICES=On \
 -DENABLE_HTTP=On

cd build
sudo make -j$(nproc)
make -j$(nproc) check
make install
```

## Configuration and Running

```bash
mkdir /opt/valhalla/valhalla && cd $_
mkdir valhalla_tiles && mkdir conf
sudo apt-get install -y curl jq unzip spatial-bin
sudo apt-get install -y curl jq unzip spatialite-bin

git clone https://github.com/valhalla/valhalla.git ~/valhalla_git/
cp -r ~/valhalla_git/scripts/ /opt/valhalla/valhalla/scripts
rm -rf ~/valhalla_git

curl -O https://download.geofabrik.de/asia/bangladesh-latest.osm.pbf

valhalla_build_config --mjolnir-tile-dir ${PWD}/valhalla_tiles --mjolnir-tile-extract ${PWD}/valhalla_tiles.tar --mjolnir-timezone ${PWD}/valhalla_tiles/timezones.sqlite --mjolnir-admin ${PWD}/valhalla_tiles/admins.sqlite > conf/valhalla.json

valhalla_build_admins --config conf/valhalla.json bangladesh-latest.osm.pbf

./scripts/valhalla_build_timezones conf/valhalla.json

valhalla_build_tiles -c conf/valhalla.json bangladesh-latest.osm.pbf
find valhalla_tiles | sort -n | tar -cf "valhalla_tiles.tar" --no-recursion -T -

valhalla_build_tiles -c conf/valhalla.json roads.pbf
find valhalla_tiles | sort -n | tar -cf "valhalla_tiles.tar" --no-recursion -T -

screen -dmS Valhalla
screen -S Valhalla -p 0 -X stuff 'cd /opt/valhalla/valhalla && nvm use 10.16.2 && valhalla_service /opt/valhalla/valhalla/conf/valhalla.json 1\n'

#valhalla_service /opt/valhalla/valhalla/conf/valhalla.json 1

valhalla_service /opt/bkash-geo-engines/navigation-engine/navigation-data/conf/valhalla.json 1
```

## Pelias

#### Elasticsearch

Version 7.8.0

- Version : **7.8.6**
- Version Dependency : **Must**
- Tested Platform : **Ubuntu 18.04**

##### Install Java

```bash
mkdir -p /opt/java
cd /opt/java

sudo apt-get -y purge openjdk-\*
curl -O -J -L https://eu2.contabostorage.com/ecf3feb4159740a4bd937c3d68be7234:dl.locusq/jdk-8u231-linux-x64.tar.gz
tar xzvf jdk-8u231-linux-x64.tar.gz
sudo update-alternatives --install /usr/bin/java java /opt/java/jdk1.8.0_231/bin/java 100
sudo update-alternatives --config java
sudo update-alternatives --install /usr/bin/javac javac /opt/java/jdk1.8.0_231/bin/javac 100
sudo update-alternatives --config javac
sudo update-alternatives --install /usr/bin/jar jar /opt/java/jdk1.8.0_231/bin/jar 100
sudo update-alternatives --config jar

sudo bash -c "echo 'JAVA_HOME=/opt/java/jdk1.8.0_231' >> /etc/profile"
sudo bash -c "echo 'JRE_HOME=/opt/java/jdk1.8.0_231/jre' >> /etc/profile"
sudo bash -c "echo 'export JAVA_HOME' >> /etc/profile"
sudo bash -c "echo 'export JRE_HOME' >> /etc/profile"
source /etc/profile
rm /opt/java/jdk-8u231-linux-x64.tar.gz
```

Now Install Elasticsearch

```bash
mkdir -p /opt/elasticsearch
cd /opt/elasticsearch

curl -O -J -L https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.7.0-linux-x86_64.tar.gz
curl -O -J -L https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.7.0-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-7.7.0-linux-x86_64.tar.gz.sha512
tar -xzf elasticsearch-7.7.0-linux-x86_64.tar.gz
cd elasticsearch-7.7.0/

/opt/elasticsearch/elasticsearch-7.7.0/bin/elasticsearch-plugin install analysis-icu
```

##### Sqlite

```bash
sudo apt-get update
sudo apt-get install python sqlite3 libsqlite3-dev
```

##### Libpostal

```bash
sudo apt-get install curl autoconf automake libtool pkg-config

git clone https://github.com/openvenues/libpostal
cd libpostal
./bootstrap.sh
./configure --datadir=/opt/pelias/libpostal
make -j4
sudo make install
```

On Linux it's probably a good idea to run

```bash
sudo ldconfig
pkg-config --cflags libpostal             
pkg-config --libs libpostal               
pkg-config --cflags --libs libpostal     
```

##### Go

```bash
wget -c https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local
sudo bash -c "echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile"
source /etc/profile
```

##### go-whosonfirst-libpostal

```bash
git clone https://github.com/whosonfirst/go-whosonfirst-libpostal 
cd go-whosonfirst-libpostal
make bin 
screen -dmS LibPostal -s /bin/bash
screen -S LibPostal -p 0 -X stuff 'cd /opt/pelias/go-whosonfirst-libpostal && ./bin/wof-libpostal-server -port 3101\n'
```

### Pelias Libraries Installation

Create SH file

```bash
mkdir pelias 
cd pelias 
touch pelias_library_install.sh 
chmod 775 pelias_library_install.sh 
sudo nano pelias_library_install.sh
```

Put the following code in the pelias_library_install.sh file

```bash
#!/bin/bash

for repository in schema whosonfirst geonames openaddresses openstreetmap polylines api placeholder interpolation pip-service csv-importer; do

git clone https://github.com/pelias/${repository}.git # clone from Github

pushd $repository > /dev/null       # switch into importer directory
npm install                         # install npm dependencies
popd > /dev/null                    # return to code directory
done
```

Run the pelias_library_install.sh

```bash
./pelias_library_install.sh
```

### Data Preparation

##### Pelias Config File

Pelias runs based on a config file where every configuration is written in a json file. Usually Pelias search the file the user’s home directory

```bash
cd ~ 
touch pelias.json 
chmod 775 pelias.json
sudo nano pelias.json
```

Now insert the following configurations in the pelias.json file

```json
{
  "esclient": {
    "apiVersion": "7.7",
    "keepAlive": true,
    "requestTimeout": "120000",
    "hosts": [{
      "env": "development",
      "protocol": "http",
      "host": "localhost",
      "port": 3105
    }],
    "log": [{
      "type": "stdio",
      "json": false,
      "level": [ "error", "warning" ]
    }]
  },
  "elasticsearch": {
    "settings": {
      "index": {
        "number_of_replicas": "0",
        "number_of_shards": "5",
        "refresh_interval": "1m"
      }
    }
  },
  "interpolation": {
    "client": {
      "adapter": "null"
    }
  },
  "dbclient": {
    "statFrequency": 10000
  },
  "api": {
    "accessLog": "common",
    "textAnalyzer": "libpostal",
    "host": "http://pelias.mapzen.com/",
    "indexName": "pelias",
    "version": "1.0",
    "targets": {
      "auto_discover": false,
      "canonical_sources": ["whosonfirst", "openstreetmap", "openaddresses", "geonames", "csv"],
      "layers_by_source": {
        "openstreetmap": [ "address", "venue", "street" ],
        "openaddresses": [ "address" ],
        "geonames": [
          "country", "macroregion", "region", "county", "localadmin", "locality", "borough",
          "neighbourhood", "venue"
        ],
        "whosonfirst": [
          "continent", "empire", "country", "dependency", "macroregion", "region", "locality",
          "localadmin", "macrocounty", "county", "macrohood", "borough", "neighbourhood",
          "microhood", "disputed", "venue", "postalcode", "continent", "ocean", "marinearea"
        ],
        "csv": ["poi"]
      },
      "source_aliases": {
        "osm": [ "openstreetmap" ],
        "oa":  [ "openaddresses" ],
        "gn":  [ "geonames" ],
        "wof": [ "whosonfirst" ],
        "csv": [ "csv" ]
      },
      "layer_aliases": {
        "coarse": [
          "continent", "empire", "country", "dependency", "macroregion", "region", "locality",
          "localadmin", "macrocounty", "county", "macrohood", "borough", "neighbourhood",
          "microhood", "disputed", "postalcode", "continent", "ocean", "marinearea", "poi"
        ]
      }
    },
    "services": {
      "placeholder": {
        "url": "http://localhost:3103"
      },
      "libpostal": {
        "url": "http://localhost:3101"
      },
      "pip": {
        "url": "http://localhost:3102"
      },
      "interpolation": {
        "url": "http://localhost:3104"
      }
    }
  },
  "schema": {
    "indexName": "pelias"
  },
  "logger": {
    "level": "debug",
    "timestamp": true,
    "colorize": true
  },
  "acceptance-tests": {
    "endpoints": {
      "local": "http://localhost:3100/v1/",
      "dev-cached": "http://pelias.dev.mapzen.com.global.prod.fastly.net/v1/",
      "dev": "http://pelias.dev.mapzen.com/v1/",
      "prod": "http://search.mapzen.com/v1/",
      "prod-uncached": "http://pelias.mapzen.com/v1/",
      "prodbuild": "http://pelias.prodbuild.mapzen.com/v1/"
    }
  },
  "imports": {
    "adminLookup": {
      "enabled": true,
      "maxConcurrentRequests": 100,
      "usePostalCities": false
    },
    "blacklist": {
      "files": []
    },
    "csv": {
      "datapath": "/opt/pelias/csv-importer/data",
      "files": ["pelias_bkash_agent.csv"],
      "download": [

      ]
    },
    "geonames": {
      "datapath": "/opt/pelias/data",
      "countryCode": "BD",
      "sourceURL": "https://download.geonames.org/export/dump/BD.zip"
    },
    "openstreetmap": {
      "datapath": "/opt/pelias/data",
      "leveldbpath": "/tmp",
      "import": [{
        "filename": "bangladesh-latest.osm.pbf"
      }]
    },
    "whosonfirst": {
      "datapath": "/opt/pelias/data",
      "sqlite":true
    }
  }
}
```

Need to change according to server and other configurations

##### Pelias Schema

```bash
cd /opt/pelias/schema
./bin/create_index
```

for droping schema (It will erase all data)

```bash
node scripts/drop_index.js      # it will ask for confirmation first
```

##### WhosOnFirst

Download WhosOnFirst Data from the following site. https://dist.whosonfirst.org/sqlite/ Create a folder named data within whosonfirst folder and download the data

```bash
cd /opt/pelias
mkdir data
wget https://dist.whosonfirst.org/sqlite/whosonfirst-data-admin-bd-latest.db.bz2
mkdir sqlite
bzip2 -d whosonfirst-data-admin-bd-latest.db.bz2
mv whosonfirst-data-admin-bd-latest.db whosonfirst-data-latest.db
mv whosonfirst-data-latest.db sqlite/
```

Make sure in your configuration you showed the right path of the file. In pelias.json file

```json
    ...
    "whosonfirst": {
      "datapath": "/opt/pelias/data",
      "sqlite":true
    }
    ...
```

Now run `WhosOnFirst`. Make sure the go-whosonfirst-libpostal server is running

```bash
npm start
```

##### Pelias PIP (Point in polygon) Service

```bash
cd /opt/pelias/pip-service
screen -dmS PIP -s /bin/bash
screen -S PIP -p 0 -X stuff 'cd /opt/pelias/pip-service && nvm use 10.16.2 && PORT=3102 npm start\n'
```

##### Pelias Placeholder Service

Download the pelias placeholder sqlite file in the data folder of placeholder from the link https://s3.amazonaws.com/pelias-data.nextzen.org/placeholder/store.sqlite3.gz

```bash
cd /opt/pelias/placeholder/
mkdir data
cd data
wget https://s3.amazonaws.com/pelias-data.nextzen.org/placeholder/store.sqlite3.gz
gunzip store.sqlite3.gz
cd ..
```

Start Placeholder

```bash
screen -dmS Placeholder -s /bin/bash
screen -S Placeholder -p 0 -X stuff 'cd /opt/pelias/placeholder/ && PORT=3103 npm start\n'
```

##### Pelias Interpolation

Interpolation service can be ignored if no address in needed with road no. If need follow the documentation [GitHub - pelias/interpolation: global street address interpolation service (beta)](https://github.com/pelias/interpolation) ***(Currently ignored)*** After running all the services the pelias.json config file should be like

```json
...
"services": {
    "placeholder": {
        "url": "http://localhost:3101"
    },
    "libpostal": {
        "url": "http://localhost:3103"
    },
    "pip": {
        "url": "http://localhost:3102"
    },
    "interpolation": {
        "url": "http://localhost:3104"
    }
}
...
```

##### Pelias geonames

Create a data directory in pelias folder. Download the Geonames of Bangladesh from the following site. https://download.geonames.org/export/dump/BD.zip Need to declare following in pelias.json config file

```json
...
"geonames": {
    "datapath": "/opt/pelias/data",
    "countryCode": "BD",
    "sourceURL": "https://download.geonames.org/export/dump/BD.zip"
},
...
```

Insert Geonames in elasticsearch

```bash
cd /opt/pelias/geonames
npm start
```

It will take time to insert data

##### Pelias openstreetmaps

In the pelias data directory download the latest Bangladesh pbf file from geofabrik http://download.geofabrik.de/asia/bangladesh-latest.osm.pbf

```bash
cd /opt/pelias/data
wget http://download.geofabrik.de/asia/bangladesh-latest.osm.pbf
```

Need to declare following in pelias.json config file

```json
...
"openstreetmap": {
    "datapath": "/opt/pelias/data",
    "leveldbpath": "/tmp",
    "import": [{
        "filename": "bangladesh-latest.osm.pbf"
    }]
},
...
```

Now Insert OSM data to elasticsearch

```bash
npm start
```

##### Pelias csv-importer

In the pelias data directory put the pre-formatted csv file in the following format

| id  | name | source | layer | lat | lon | addendum_json_data |
| --- | --- | --- | --- | --- | --- | --- |
| 1   | POI name | custom | poi | 23.45 | 92.34 | "{ ""id"": 600 }" |

This importer can be configured in pelias-config, in the imports.csv hash. A sample configuration file might look like

```json
...
"csv": {
  "datapath": "/opt/pelias/csv-importer/data",
  "files": ["imports.csv"],
  "download": []
},
...
```

You must put any custom source and layers imported by your data in pelias.json

```json
...
{
  "api": {
    "targets": {
      "csv": ["poi"]
    }
  }
}
...
```

Now start insert custom data to elasticsearch

```bash
cd /opt/pelias/csv-importer
./bin/start
```

It will take time to insert data

### Pelias Deployment

Make sure elasticsearch, libpostal and four pelias services up and running in configured port.

##### Pelias API

```bash
cd /opt/pelias/api 
screen -dmS GeoCodingAPI -s /bin/bash
screen -S GeoCodingAPI -p 0 -X stuff 'cd /opt/pelias/api && nvm use 10.16.2 && PORT=3100 npm start\n'
```

Now the 6 API endpoints can provide geocoding data according to given parameter