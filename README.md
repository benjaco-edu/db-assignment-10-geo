# DB Assignment 10 geo data

https://github.com/datsoftlyngby/soft2019spring-databases/blob/master/assignments/assignment10.md


## Get up and running

Following will download the data, and create a Docker container.

You will be put inside the docker container where you have to execute  all the following commands

**Start container and install dependencies to fetch the database**
```
sudo docker run --rm --name my_mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=pass1234 -d mysql
sudo docker exec -it my_mysql bash
echo "Following is inside the container"
apt-get update
apt-get install wget -y
```
**Download the data**
```
wget https://raw.githubusercontent.com/benjaco-edu/db-assingment-10-geo/master/data/cykelstativ.sql
wget https://raw.githubusercontent.com/benjaco-edu/db-assingment-10-geo/master/data/gadetraer.sql
wget https://raw.githubusercontent.com/benjaco-edu/db-assingment-10-geo/master/data/parkregister.sql
wget https://raw.githubusercontent.com/benjaco-edu/db-assingment-10-geo/master/data/tungvognsnet.sql
wget https://raw.githubusercontent.com/benjaco-edu/db-assingment-10-geo/master/data/udsatte_byomraader.sql
```
**The rest will be executed inside of the mysql bash**

```
mysql -u root -ppass1234
```

**Import data**
```sql
create database cph DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

use cph;

SET autocommit=0 ;
source cykelstativ.sql;
source gadetraer.sql;
source parkregister.sql;
source tungvognsnet.sql;
source udsatte_byomraader.sql;

ALTER TABLE cykelstativ ADD SPATIAL INDEX(wkb_geometry);
ALTER TABLE gadetraer ADD SPATIAL INDEX(wkb_geometry);
ALTER TABLE parkregister ADD SPATIAL INDEX(wkb_geometry);
ALTER TABLE tungvognsnet ADD SPATIAL INDEX(wkb_geometry);
ALTER TABLE udsatte_byomraader ADD SPATIAL INDEX(wkb_geometry);

COMMIT ;
SET autocommit=1;
```


## Queries

### How many parks are located in exposed areas?

_Query_

```sql
with parkregister_extended as (select *, ST_Area(wkb_geometry) as realm2 from parkregister)

select byomraade,
       delomraade,
       count(parkregister_extended.areal_id)  as "no_of_parks",
       sum(parkregister_extended.realm2)      as "overall_park_area",
       sum(parkregister_extended.realm2) / m2 as "percent_park_area"
from udsatte_byomraader,
     parkregister_extended
where ST_Within(parkregister_extended.wkb_geometry, udsatte_byomraader.wkb_geometry)
group by udsatte_byomraader.id
order by percent_park_area desc;
```

I add the number of parks per square kilometer, and the percent of the area which is park which I think is more because it takes care of big parks, the number can be a little screwed if the park only touches the area. I could not find any function to say how much of the area there where overlapping.

_Result_

```
+-------------------+--------------------+-------------+--------------------+----------------------+
| byomraade         | delomraade         | no_of_parks | overall_park_area  | percent_park_area    |
+-------------------+--------------------+-------------+--------------------+----------------------+
| Nordvest/Ryparken | Ryparken           |           7 |  506856.3909881475 |   0.4451962633218131 |
| Valby/Sydhavnen   | Sydhavnen          |           9 | 1133586.1791762048 |   0.4439245425501555 |
| Nørrebro          | Ved Bispeengbuen   |           3 |  40938.23739301319 |  0.20124289004415927 |
| Valby/Sydhavnen   | Ved Kulbanevej     |           1 |  49297.82090760503 |  0.09428241011716978 |
| Nørrebro          | Ved Jagtvej        |           2 |  9620.596170289327 |  0.07672110313874596 |
| Nørrebro          | Ydre Nørrebro      |           1 |  68123.62915523286 |   0.0692643436304833 |
| Valby/Sydhavnen   | Ved Valby Langgade |           1 | 12753.542520959758 |  0.06260206220651351 |
| Amager/Sundby     | Urbanplanen mv.    |           1 | 63494.482850991335 |  0.05993556871619254 |
| Nørrebro          | Indre Nørrebro     |           2 | 23069.796054819122 | 0.055551720035202624 |
| Nordvest/Ryparken | Nordvest           |          11 | 133141.05108804916 | 0.050309339570643055 |
| Tingbjerg/Husum   | Tingbjerg/Husum    |           5 |  47107.07675534113 |  0.02121078586778756 |
| Valby/Sydhavnen   | Ved Folehaven      |           1 | 7786.3333955716735 | 0.013810550971577793 |
+-------------------+--------------------+-------------+--------------------+----------------------+

```


### How many trees are located in exposed areas?

_Query_

```sql
select byomraade,
       delomraade,
       count(gadetraer.id)  as "no_of_trees",
       count(gadetraer.id) / (m2 / 1000000) as "no_of_trees_per_m2"
from udsatte_byomraader, gadetraer
where st_within(gadetraer.wkb_geometry, udsatte_byomraader.wkb_geometry)
group by udsatte_byomraader.id
order by no_of_trees_per_m2 desc;
```

Again, it is up against the size of the area

_Result_

```
+-------------------+--------------------------+-------------+--------------------+
| byomraade         | delomraade               | no_of_trees | no_of_trees_per_m2 |
+-------------------+--------------------------+-------------+--------------------+
| Nørrebro          | Ved Bispeengbuen         |         260 |          1278.2694 |
| Nørrebro          | Ved Jagtvej              |          62 |           940.8194 |
| Amager/Sundby     | Ved Frankrigsgade        |          55 |           627.8539 |
| Nørrebro          | Ved Jagtvej              |          74 |           590.1116 |
| Valby/Sydhavnen   | Sydhavnen                |        1340 |           524.7494 |
| Nordvest/Ryparken | Ryparken                 |         570 |           500.6588 |
| Nørrebro          | Indre Nørrebro           |         176 |           423.7900 |
| Amager/Sundby     | Urbanplanen mv.          |         318 |           300.1699 |
| Nordvest/Ryparken | Ved Bispebjerg Parkallé  |          65 |           297.7554 |
| Nordvest/Ryparken | Nordvest                 |         721 |           272.4456 |
| Valby/Sydhavnen   | Ved Kulbanevej           |         142 |           271.5624 |
| Tingbjerg/Husum   | Tingbjerg/Husum          |         600 |           270.1607 |
| Valby/Sydhavnen   | Ved Valby Langgade       |          54 |           265.0957 |
| Valby/Sydhavnen   | Ved Folehaven            |         145 |           257.1834 |
| Nørrebro          | Ydre Nørrebro            |         247 |           251.1439 |
| Amager/Sundby     | Ved Gyldenrisvej         |          31 |           106.0917 |
| Nordvest/Ryparken | Ved Bispebjerg Parkallé  |           6 |            36.6077 |
+-------------------+--------------------------+-------------+--------------------+

```

### How many bike racks are places along routes for heavy traffic?


_Query_

```sql
with heavy_roads as (select ST_GeomFromText(ST_ASTEXT(ST_Buffer(ST_GeomFromText(ST_AsText(wkb_geometry), 0), 0.00025)), 4326) as area, id, vej from tungvognsnet)
select
       count(*) as "no_racks",
       sum(cykelstativ.antal_pladser) as "no_spaces"
from heavy_roads, cykelstativ
where st_within(cykelstativ.wkb_geometry, heavy_roads.area);
```

Here, I have queried the number of bike racks and individual bike spaces within 25 meters of the center of a heavy road.

A lot of the buffer and distance functions wasn't supported for lines on a sphere, so I casted it to a flat map, modified the geo area, and casted it back, the loss in precision is next to nothing accordingly to my testing, At least for our area of the globe.

_Result_

```
+----------+-----------+
| no_racks | no_spaces |
+----------+-----------+
|      642 |      9141 |
+----------+-----------+
```

## Cleanup

Exit the shell with `CTRL + d`

Remove the docker container with `sudo docker rm -f my_mysql`
