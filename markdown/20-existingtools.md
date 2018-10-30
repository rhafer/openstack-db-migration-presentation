<!-- .slide: data-state="section-break" id="section-break-1" data-timing="10s" -->
# Let's just migrate. How hard can it be?


<!-- .slide: data-state="normal" id="dump-reload" data-timing="20s" data-menu-title="dump-reload" -->
# It's all just SQL!

### ... or why a dump and reload approach won't work

* SQL dialects
* Type systems
* some project used slightly different schema depending on database backend


<!-- .slide: data-state="section-break-3" id="existing-tools" data-timing="20s" data-menu-title="" -->
# Somebody must have done this before

## Looking for existing tools


<!-- .slide: data-state="normal" id="requirements" data-timing="20s" data-menu-title="requirements" -->
# Requirements

* Migrate OpenStack data from PostgreSQL to MariaDB
* The Database Scheme on the target side is created by the OpenStack tools (db_sync)
* Some downtime is acceptable
* Cope with/detect incompatibilities in Data
* Focus on getting a working and reliable migration
  * Optimizations, e.g. to decrease downtime can happen later.

### Nice to have
* The migration tool should be able to skip certain tables during the migrations
  * E.g. those that contain meta data about alembic and sqlalchemy-migrate. As they are written by the db_sync tools
* Possibility to skip "soft-deleted" rows

Note:
* Some OpenStack projects use "soft-deletes" (e.g. cinder and nova) a lot)


<!-- .slide: data-state="normal" id="pg2mysql" data-timing="20s" data-menu-title="pg2mysql" -->
# pg2mysql

https://github.com/pivotal-cf/pg2mysql

* Implements a one-time online migration
* written in GO
* does some schema validation before migration
  * checking for the field length and type compatibility
  * but doesn't care about charset issues
* Issues with quoting/escaping
  * doesn't handle tables using PostgresSQL reserved words in
    their names (like "user" and "group" in keystone)
  * We'd need to address all these issues.
* No possibilty to skip tables (e.g. alembic and sqlalchemy-migrate related stuff).
* Doesn't handle enum collumns correctly (i.e. calls length() on them)
* In general seems to be very bare bones


<!-- .slide: data-state="normal" id="kitchensync" data-timing="20s" data-menu-title="kitchensync" -->
# kitchensync

https://github.com/willbryant/kitchen_sync
* provides online incremental synchronization between databases
  * The "incremental" sync has certain requirements on the database schema
    which some OpenStack database don't fulfill
* written in C++, has very little external dependencies (just the postgresql
  and mariadb client libs)
* pretty strict expectations for type mapping between source and target
  database (might not be an issue for openstack)
* can't handle enum type columns (similar to pg2mysql)
* very hard to debug (poor logging)


<!-- .slide: data-state="normal" id="kitchensync" data-timing="20s" data-menu-title="othertools" -->
## Other?
* More tools exist
* Some commercial offerings (non OpenSource)
* Some require control over Schema in the target database
* ad-hoc special purpose scripts for a specific use case (i.e. for
  a certain application)

### none of the existing tools fulfilled our requirements. And now?
