<!-- .slide: data-state="section-break" id="existing-tools" data-timing="20s" data-menu-title="" -->
# Somebody must have done this before
## What about existing tools?


<!-- .slide: data-state="normal" id="requirements" data-timing="180s" data-menu-title="requirements" -->
# Requirements
* Who creates the Schema?
<!-- .element: class="fragment" -->
* What about Downtime?
<!-- .element: class="fragment" -->
* How are incompatibilities handled?
<!-- .element: class="fragment" -->

### And other things that are nice to have

Note:
Requirements:
* Migrate OpenStack data from PostgreSQL to MariaDB
* The Database Scheme on the target side is created by the OpenStack tools (db_sync)
* Some downtime is acceptable
* Cope with/detect incompatibilities in Data
* Reliabilty over speed

Nice to have:
* The migration tool should be able to skip certain tables during the migrations
  * E.g. those that contain meta data about alembic and sqlalchemy-migrate. As they are written by the db_sync tools
* Possibility to skip "soft-deleted" rows
* Some OpenStack projects use "soft-deletes" (e.g. cinder and nova) a lot)
* incremental sync


<!-- .slide: data-state="normal" id="pg2mysql" data-timing="180s" data-menu-title="pg2mysql" -->
# Exiting tools

* pg2mysql (https://github.com/pivotal-cf/pg2mysql) 
* kitchensync (https://github.com/willbryant/kitchen_sync)
* Other tools

Note:
* pg2mysql:
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
* kitchensync
  * provides online incremental synchronization between databases
    * The "incremental" sync has certain requirements on the database schema
      which some OpenStack database don't fulfill
  * written in C++, has very little external dependencies (just the postgresql
    and mariadb client libs)
  * pretty strict expectations for type mapping between source and target
    database (might not be an issue for openstack)
  * can't handle enum type columns (similar to pg2mysql)
  * very hard to debug (poor logging)
* other
  * More tools exist
  * Some commercial offerings (non OpenSource)
  * Some require control over Schema in the target database
  * ad-hoc special purpose scripts for a specific use case (i.e. for
    a certain application)


<!-- .slide: data-state="normal" id="wrong-tool" data-timing="20s" data-menu-title="othertools" -->
## None of the existing tools fits our needs
<img class="full-slide" src="images/hammer_bent_screw.jpg"/>
