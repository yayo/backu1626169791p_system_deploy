## PostGIS安装指导手册
              
### 标签            
PostGIS , OpenStreetMap , osm2pgsql    
              
----            
              
## 背景      
PostGIS 是 PostgreSQL 关系数据库的空间操作扩展。它为 PostgreSQL 提供了存储、查询和修改空间关系的能力。
OpenStreetMap是一个网上地图协作计划，其以类似于维基百科的运作模式，目标是创造一个内容自由且能让所有人编辑的世界地图。
      
## 环境    
1\. CentOS 7 x64    
2\. PostgreSQL 9.6    
3\. PostGIS 2.3.3    
4\. 样本数据来自OpenStreetMap公开的cn.pbf中国的地理位置信息数据。      
    
## 部署PostgreSQL 9.6    
https://www.postgresql.org/    
    
```    
$ curl -k -O https://ftp.postgresql.org/pub/source/v9.6.3/postgresql-9.6.3.tar.bz2
$ tar -xjf postgresql-9.6.3.tar.bz2
$ cd postgresql-9.6.3
$ ./configure --prefix=/usr --build=x86_64-pc-linux-gnu --host=x86_64-pc-linux-gnu --mandir=/usr/share/man --infodir=/usr/share/info --datadir=/usr/share --sysconfdir=/etc --localstatedir=/var/lib --docdir=/usr/share/doc/postgresql-9.6 --htmldir=/usr/share/doc/postgresql-9.6/html --libdir=/usr/lib64/postgresql-9.6/lib64 --prefix=/usr/lib64/postgresql-9.6 --datadir=/usr/share/postgresql-9.6 --docdir=/usr/share/doc/postgresql-9.6 --includedir=/usr/include/postgresql-9.6 --mandir=/usr/share/postgresql-9.6/man --sysconfdir=/etc/postgresql-9.6 --with-system-tzdata=/usr/share/zoneinfo --enable-spinlocks --enable-integer-datetimes --enable-thread-safety --without-gssapi --without-ldap --with-pam --with-perl --with-python --with-readline --with-openssl --without-systemd --without-tcl --with-uuid=e2fs --with-libxml --with-libxslt --with-zlib --disable-nls
$ gmake world -j 32    
$ gmake install-world    
```    
      
环境变量配置      
```    
$ cat  ~/env_pg.sh    
export LANG=en_US.utf8
export PGPORT=1921
export PGDATA=/data01/pgdata/pg_root_96
export PGHOME=/usr/lib64/postgresql-9.6
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH
export PATH=$PGHOME/bin:$PATH:.
export PGDATABASE=postgres

$ . ~/env_pg.sh    
```    
      
## PostGIS 部署    
依赖软件包安装
    
### geos    
http://trac.osgeo.org/geos    
```    
$ cd ~    
$ curl -O http://download.osgeo.org/geos/geos-3.5.1.tar.bz2    
$ tar -xjf geos-3.5.1.tar.bz2     
$ cd geos-3.5.1
$ ./configure --prefix=/home/postgres/geos    
$ make -j 32    
$ make install    
```    
      
### proj    
https://trac.osgeo.org/proj/    
```    
$ cd ~    
$ curl -O http://download.osgeo.org/proj/proj-4.9.3.tar.gz    
$ tar -xzf proj-4.9.3.tar.gz    
$ cd proj-4.9.3    
$ ./configure --prefix=/home/postgres/proj4    
$ make -j 32    
$ make install    
```    
    
### GDAL    
http://gdal.org/    
    
```    
$ cd ~    
$ curl -O http://download.osgeo.org/gdal/2.2.1/gdal-2.2.1.tar.gz    
$ tar -xzf gdal-2.2.1.tar.gz    
$ cd gdal-2.2.1    
$ ./configure --prefix=/home/postgres/gdal --with-pg=/usr/lib64/postgresql-9.6/bin/pg_config
$ make -j 32    
$ make install    
```    
      
### cgal    
http://www.cgal.org/download.html    
```    
$ cd ~    
$ curl -k -L -O https://github.com/CGAL/cgal/releases/download/releases%2FCGAL-4.10/CGAL-4.10.tar.xz
$ tar -xJf CGAL-4.10.tar.xz  
$ cd CGAL-4.10
$ mkdir build    
$ cd build    
$ cmake -D CMAKE_INSTALL_PREFIX=/home/postgres/cgalhome ../    
$ make -j 32    
$ make install    
```    
    
    
### postgis 2.3.3    
http://postgis.net/source/    
    
库      
```    
# cat /etc/ld.so.conf    
/usr/lib64/postgresql-9.6/lib64
/home/postgres/geos/lib    
/home/postgres/proj4/lib    
/home/postgres/gdal/lib    
/home/postgres/cgalhome/lib64
    
# ldconfig    
```    
    
部署PostGIS    
```    
$ curl -O http://download.osgeo.org/postgis/source/postgis-2.3.3.tar.gz    
    
$ tar -xzf postgis-2.3.3.tar.gz    
$ cd postgis-2.3.3    
$ ./configure --prefix=/home/postgres/postgis --with-gdalconfig=/home/postgres/gdal/bin/gdal-config --with-pgconfig=/usr/lib64/postgresql-9.6/bin/pg_config --with-geosconfig=/home/postgres/geos/bin/geos-config --with-projdir=/home/postgres/proj4
$ make -j 32    
$ make install    
```    
    
    
## 初始化数据库集群    
    
```    
# useradd --no-create-home postgres
# mkdir -p /run/postgresql $PGDATA
# chown postgres /run/postgresql $PGDATA
# su postgres -c "$PGHOME/bin/initdb -D $PGDATA -E UTF8 --locale=C -U postgres"
```    
    
启动数据库集群    
```    
# su postgres -c "$PGHOME/bin/pg_ctl start"
```    
## 安装插件    
安装PostGIS插件后，PostgreSQL才具备GIS功能。
```    
$ psql -U postgres -d postgres    
psql (9.6)    
Type "help" for help.    
postgres=# create extension postgis;    
CREATE EXTENSION    
postgres=# create extension fuzzystrmatch;    
CREATE EXTENSION    
postgres=# create extension postgis_tiger_geocoder;    
CREATE EXTENSION    
postgres=# create extension postgis_topology;    
CREATE EXTENSION    
postgres=# create extension address_standardizer;    
CREATE EXTENSION    
```    
    
## 安装osm2pgsql    
我们需要借助osm2pgsql将OpenStreetMap数据导入PostGIS    
http://wiki.openstreetmap.org/wiki/Osm2pgsql    
https://github.com/openstreetmap/osm2pgsql    
    
```    
# curl -k -L -O https://github.com/openstreetmap/osm2pgsql/archive/0.90.2.tar.gz
# tar -xzf 0.90.2.tar.gz
# cd osm2pgsql-0.90.2
# mkdir build && cd build
# cmake -DCMAKE_INSTALL_PREFIX=/usr -DGEOS_INCLUDE_DIR=/home/postgres/geos/include -DGEOS_LIBRARY=/home/postgres/geos/lib/libgeos.so -DPROJ_INCLUDE_DIR=/home/postgres/proj4/include -DPROJ_LIBRARY=/home/postgres/proj4/lib/libproj.so ..
# make && make install
```    
    
安装目标    
```    
# ls -1 /usr/bin/osm2pgsql /usr/share/man/man1/osm2pgsql.1 /usr/share/osm2pgsql/default.style /usr/share/osm2pgsql/empty.style /usr/share/osm2pgsql/900913.sql
```    
    
## 样本数据下载    
OpenStreetMap是一个开放的GIS信息数据共享库。      
https://www.openstreetmap.org    
    
可以从镜像站点下载共享的pbf数据    
http://download.gisgraphy.com/openstreetmap/pbf/
    
下载中国的PBF数据    
```    
$ curl -O http://download.gisgraphy.com/openstreetmap/pbf/CN.tar.bz2    
-rw-r--r-- 1 root root 279M Nov 25  2016 CN.tar.bz2
$ tar -C /data01 -xjf CN.tar.bz2
-rw-r--r-- 1 root root 278M Nov 25  2016 /data01/CN
```    
    
## 导入样本数据    
```    
$ psql -U postgres -d postgres
postgres=# \dx    
                                                                    List of installed extensions    
          Name          | Version |   Schema   |                                                     Description                                                         
------------------------+---------+------------+---------------------------------------------------------------------------------------------------------------------    
 address_standardizer   | 2.3.3   | public     | Used to parse an address into constituent elements. Generally used to support geocoding address normalization step.    
 fuzzystrmatch          | 1.1     | public     | determine similarities and distance between strings    
 plpgsql                | 1.0     | pg_catalog | PL/pgSQL procedural language    
 postgis                | 2.3.3   | public     | PostGIS geometry, geography, and raster spatial types and functions    
 postgis_tiger_geocoder | 2.3.3   | tiger      | PostGIS tiger geocoder and reverse geocoder    
 postgis_topology       | 2.3.3   | topology   | PostGIS topology spatial types and functions    
(6 rows)    
```    
      
使用osm2pgsql将中国地理数据导入PostgreSQL数据库    
```    
$ export PGPASS=postgres    
    
$ osm2pgsql -H 127.0.0.1 -P 1921 -U postgres -d postgres -c -l -C 2000 --number-processes 8 -p yayo -r pbf /data01/CN    
```    
      
数据分为点，线段，道路，区域。      
```    
postgres=# \dt+    
                            List of relations    
  Schema  |      Name       | Type  |  Owner   |    Size    | Description     
----------+-----------------+-------+----------+------------+-------------    
 public   | yayo_line     | table | postgres | 738 MB     |     
 public   | yayo_point    | table | postgres | 80 MB      |     
 public   | yayo_polygon  | table | postgres | 334 MB     |     
 public   | yayo_roads    | table | postgres | 312 MB     |     
    
postgres=# \di+    
                                                     List of relations    
  Schema  |                      Name                       | Type  |  Owner   |      Table      |    Size    | Description     
----------+-------------------------------------------------+-------+----------+-----------------+------------+-------------    
 public   | yayo_line_index                               | index | postgres | yayo_line     | 193 MB     |     
 public   | yayo_point_index                              | index | postgres | yayo_point    | 46 MB      |     
 public   | yayo_polygon_index                            | index | postgres | yayo_polygon  | 90 MB      |     
 public   | yayo_roads_index                              | index | postgres | yayo_roads    | 69 MB      |     
```    
    
## GIS数据用法举例    
http://live.osgeo.org/zh/quickstart/postgis_quickstart.html    
    
    
### 1\. 在point表中，查询城市对应的坐标    
```    
postgres=# SELECT name, ST_AsText(way), ST_AsEwkt(way), ST_X(way), ST_Y(way) FROM yayo_point where place='city' ;
....    
 无锡市 | POINT(120.2954534 31.5756347) | SRID=4326;POINT(120.2954534 31.5756347) | 120.2954534 | 31.5756347    
 青岛市 | POINT(120.3497193 36.0895093) | SRID=4326;POINT(120.3497193 36.0895093) | 120.3497193 | 36.0895093    
....    
```    
      
### 2\. 空间查询    
查询两城市间距离：
```    
SELECT p1.name,p2.name,ST_DistanceSphere(p1.way,p2.way) FROM (select * from yayo_point where place='city' and name ~ '北京') p1 , (select * from yayo_point where place='city' and name ~ '杭州') p2 where p1.name <> p2.name ;    
    
  name  |  name  | st_distance_sphere     
--------+--------+--------------------    
 北京市 | 杭州市 |   1128188.78978971    
(1 row)    
```    
    
### 3\. 区域类型查询      
```    
postgres=# select name from yayo_polygon where name ~ '杭州' ;    
               name                   
----------------------------------    
 杭州天目外国语学校    
 杭州野生动物世界    
 浙江省杭州第二中学    
 杭州邮政大楼    
....    
```    
    
### 4\. 点面判断举例    
```    
postgres=# select name from yayo_polygon where name ~ '杭州' and way ~ ST_GeomFromEWKT('SRID=4326;POINT(120.3 30.416667)') ;    
       name            
-------------------    
 杭州市 Hangzhou    
(1 row)    
```    
    
## 参考  
https://www.openstreetmap.org    
    
http://postgis.net/docs/manual-2.3/    
  
http://live.osgeo.org/zh/quickstart/postgis_quickstart.html    

