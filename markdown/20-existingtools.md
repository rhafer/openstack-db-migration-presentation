<!-- .slide: data-state="normal" id="lets-migrate" data-timing="10s" -->
# Let's just migrate.
<img class="full-slide" src="images/how-hard-can-it-be.jpg" />


<!-- .slide: data-state="normal" id="dump-reload" data-timing="20s" data-menu-title="dump-reload" -->
# It's all just SQL!

<code style="font-size: 5rem; display: block; margin: 15%">
    pg_dumpall | mysql
</code>


<!-- .slide: data-state="normal" id="sql-diff" data-timing="20s" data-menu-title="sql-diff" -->
# ... or why a simple dump and reload won't work

* Syntax differences
  * Quotes, Comments, case-sensitivity
* Data Types
  * MySQL Text Types have a size limit (TEXT can be 64kB)
  * different accuracy of default Time/Date types
  * BOOLEAN vs TINYINT

Note:
* "TEXT" is used for quite a few collumns in OpenStack
  * Mainly "description" and similar columns


<!-- .slide: data-state="normal" id="openstack-diff" data-timing="20s" data-menu-title="openstack-diff" -->
# Backend specific differences in OpenStack
### Have you ever tried this:

  `openstack volume create --size 1 "My ` &#x1F4BE;`"`

* It works with PostgreSQL <!-- .element: class="fragment" -->
* But with MySQL:<!-- .element: class="fragment" -->

<asciinema-player id="player" cols="80" rows="6" speed="3" font-size="big" theme="solarized-dark" src="images/emoji.cast"></asciinema-player>
<!-- .element: class="fragment emoji-cast" -->

Note:
* UTF8 in MySQL is not real UTF8
  * It's limited to 3 Byte UTF-8 characters
  * that reflects Unicode Codepoint 0x0000 to 0xFFFF (Basic Multilingual
    Plane)
  * It covers most commonly used characters
  * But misses Emojis, Mathematical Symbols and some less often used CJK
    characters
* If the full range is needed with MySQL you need to use "utf8mb4"
* PostgreSQL supports the full utf8 range by default
* Many OpenStack projects are still hardcoding their schema to "utf8"
* There is no good way to solve this issue programmatically
  * trying to "convert" from "real" utf8 to mysql's 3byte encode would always
  mean data loss
  * Any tool we use of the migration would need to be able to detect issues
    and provider pointers on how to fix them
* Converting every OpenStack project to use "utf8mb4" is also not easy to
  undertake. It was raised already in the past, but never really completed
  It's also unsure what this means for existing deployments.
* Characters > 0xFFFF are unlikely to appear in OpenStack database but
  we need to be aware of it for the migration


<!-- .slide: data-state="normal" id="openstack-diff-2" data-timing="20s" -->
# Backend specific differences in OpenStack
### some projects use slightly different schemas depending on database backend

Note:
* Ceilometer's workaround for high precision timestamps
  * DECIMAL+sqlalchemy magic in MySQL
  * TIMESTAMP in PostgreSQL


<!-- .slide: data-state="normal" id="operations" data-timing="20s" -->
# Operational thoughts

### Incremental Sync
* Run first DB conversion while the services are still online
* take services offline
* do another sync to just update/delete/add the rows changed since
  the last sync
* Switch services to new database

### Onetime migration
* Shutdown services
* sync/copy directly from PostgreSQL to MySQL
* reconfigure services and start
* decommision PostgreSQL

Note:
* incremental:
  * Pros:
    * minimizes Downtime
  * Cons:
    * No standard way to find added, changed, deleted Objects since last sync
      (might be possible via [https://wiki.postgresql.org/wiki/Audit_trigger])
    * A lot more complex to implement (but there might be existing tools)
* onetime:
  * Pros:
    * Probably the most straight-forward to implement
  * Longer downtime
  * Needs a live system with both databases running
  * harder to do a "dry-run" test with production data
* dump and reload:
  * Pros
    * Would more easily allows dry-runs and importing the data into some test setup outside the production env
    * Doesn't need both DBs running during the migration
  * Cons
    * Most like needs the longest downtime.
      As all data needs to written at least twice while all OpenStack services
      are down
    * What intermediate format to dump to?


<!-- .slide: data-state="section-break" id="existing-tools" data-timing="20s" data-menu-title="" -->
# Somebody must have done this before
## What about existing tools?


<!-- .slide: data-state="normal" id="requirements" data-timing="20s" data-menu-title="requirements" -->
# Requirements

### And other things that are nice to have

Note:
Requirements:
* Migrate OpenStack data from PostgreSQL to MariaDB
* The Database Scheme on the target side is created by the OpenStack tools (db_sync)
* Some downtime is acceptable
* Cope with/detect incompatibilities in Data
* Focus on getting a working and reliable migration
  * Optimizations, e.g. to decrease downtime can happen later.

Nice to have:
* The migration tool should be able to skip certain tables during the migrations
  * E.g. those that contain meta data about alembic and sqlalchemy-migrate. As they are written by the db_sync tools
* Possibility to skip "soft-deleted" rows
* Some OpenStack projects use "soft-deletes" (e.g. cinder and nova) a lot)
* incremental sync


<!-- .slide: data-state="normal" id="pg2mysql" data-timing="20s" data-menu-title="pg2mysql" -->
# pg2mysql

### https://github.com/pivotal-cf/pg2mysql

Note:
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

### https://github.com/willbryant/kitchen_sync

Note:
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
# What about other tools?

Note:
* More tools exist
* Some commercial offerings (non OpenSource)
* Some require control over Schema in the target database
* ad-hoc special purpose scripts for a specific use case (i.e. for
  a certain application)


<!-- .slide: data-state="normal" id="wrong-tool" data-timing="20s" data-menu-title="othertools" -->
## None of the existing tools fits our needs
<img class="full-slide" src="images/hammer_bent_screw.jpg"/>
