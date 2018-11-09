<!-- .slide: data-state="normal" id="break-psql2mysql" data-timing="10s" -->
# Let's write our own tool!
<img class="full-slide" src="images/how-hard-can-it-be.jpg" />


<!-- .slide: data-state="normal" id="p2m-goals" data-timing="20s" -->
# Goals

<div style="display: flex; flex-direction: row; flex-wrap: nowrap; justify-content: space-between;">
<div class="fragment" style="padding: 5px;">
    <img style="max-height:70vh; max-width: 33vw; width: auto; height: 350px" src="images/newton-logo.png">
</div>
<div class="fragment" style="padding: 5px;">
    <img style="max-height:70vh; max-width: 33vw; width: auto; height: 350px" src="images/osi_standard_logo.png">
</div>
<div class="fragment" style="padding: 5px;">
    <img style="max-height:70vh; max-width: 33vw; width: auto; height: 350px" src="images/Kiss_-_Keep_it_simple_stupid.jpg">
</div>
</div>

Note:
* We want to release this as Open Source Software to allow other to
  use/improve it.
* Newton, but should work with other (newer) OpenStack releases
  * This also means that we didn't want a completely generic tool.
    It's ok to make it somewhat specific to OpenStack databases
    Especially if it helps to keep the tool simple
* Allow to validate source data before running the actual migration and point
  the user to problematic Rows/Values


<!-- .slide: data-state="normal" id="p2m-intro" data-timing="20s" -->
# Introducing psql2mysql
https://github.com/SUSE/psql2mysql

* Written in Python (works with 2.7 and recent 3.X releases)
* Stand-alone command line tool. To migrate a single database

```
psql2mysql
    --source postgresql://<user>:<password>@<host>/<database>
    --target mysql+pymysql://<user>:<password>@<host>/<database>
    migrate
```

* Subcommands "precheck" and "purge-tables"
* Batch mode to migrate multiple databases at once

Note:
* Python was more or less the obvious choice
  * We could leverage a couple of OpenStack libraries (where it made sense)
  * We could use SQLalchemy, which provides nice abstractions to access
    different SQL databases
* Can run on any machine that has a decent network connection to the
  database(s)
  * Usually better to run it locally on one of the cluster nodes


<!-- .slide: data-state="normal" id="p2m-internals" data-timing="20s" -->
# SQLAlchemy for the win!
* Introspection
* Type Abstractions
* OpenStack is using it

Note:
SQLAlchemy provides some very useful features for our purposes
* Introspection
  * read Tables, Columns and Types
* Type Abstractions defined for most types used by OpenStack
  * "Internal" Type System that is largely backend in dependent
  * Enums are abstracted so the "enum" issue that all the tools we evaluated
    were suffering from, becomes a non-issue with SQLalchemy.
* Extensible
  * type decorators to define custom type abstractions (this helped as
    a lot. More about that on a later slide)
* If needed backend specific SQL can still be used (and we needed that)
* OpenStack is using it


<!-- .slide: data-state="section-break" id="p2m-issues-break" data-timing="20s" -->
# Issues
## ... and how we solved them


<!-- .slide: data-state="normal" id="p2m-issues-fk" data-timing="20s" -->
# Foreign keys and other constraints

Migrating table by table without a certain order will give you this:
```
IntegrityError: (pymysql.err.IntegrityError)
  (1452, u'Cannot add or update a child row: a foreign key constraint fails
    ( `neutron`.`ml2_port_bindings`, CONSTRAINT `ml2_port_bindings_ibfk_1`
      FOREIGN KEY (`port_id`) REFERENCES `ports` (`id`) ON DELETE CASCADE
    )')
```

* SQLalchemy can sort tables in order of foreign key dependency for us &#x1F603;
<!-- .element: class="fragment" data-fragment-index="1"-->
* But there might be circular dependencies &#x1F620;
<!-- .element: class="fragment" data-fragment-index="2"-->
* Luckily MySQL allows to disable constraint checks (globally and per session) &#x1F605;
<!-- .element: class="fragment" data-fragment-index="3" -->
```
SET SESSION check_constraint_checks='OFF'
SET SESSION foreign_key_checks='OFF'
```
<!-- .element: class="fragment" data-fragment-index="3"-->

Note:
* The tool migrates table by table
* During the migration foreign key constraints might be violated temporarily
* Sorting tables is not always possible (e.g. because of circular dependencies)
* When the migration is done the database should be fully consistent again


<!-- .slide: data-state="normal" id="p2m-issues-galera" data-timing="20s" -->
# Galera and huge transactions

Once the tables to migrate get a certain size you'll see this when using a
Galera cluster:

```
InternalError: (pymysql.err.InternalError)
               (1180, u'wsrep_max_ws_rows exceeded')
```

* caused by Galera limitations and non-suitable default config
* meanwhile the config defaults for galera change
* Still splitting into smaller transactions makes sense

Note:
* Galera has a hard limit on the size of single transaction, or more
  specifically "write set"
  * based on the number of rows touched
  * based on the memory used by a writeset
* psql2mysql took a very simple approach initially and issues a single
  transaction per table
* these transactions can get huge and hit those limits
* newer version of Galera/wsrep removed the default row limit and set the
  memory limit to the maximum values (about 2GB)
* Sidenote: This limits also can get issue with the ceilometer-expirer job


<!-- .slide: data-state="normal" id="p2m-issues-mem" data-timing="20s" -->
# Memory usage and runtime

<img class="full-slide fragment" src="images/memory-vs-chunksize-vs-runtime.svg"/>

Note:
* The "every table in a single transaction" approach had another issue
* Large tables cause significantly increased memory usage
  * by the migration tool itself (on the order of several GBs of memory)
  * as well as by the target mysql processes
* Solution: Multiple commits per table
  * configurable chunk-size (number of rows to commit per transcation)
* We ran a couple of tests using different chunksizes
* Took a nova database as the sample
  * biggest table had around 300 000 rows
* Python mem_profiler
* With an unlimted number of row per transcation memory usage can get huge
* With a small chunksize (1 - 100) the runtime increases significantly
* Sample database took > 30min with 1 row/commit
  vs ~3min 10 000 rows/commit
* Sweet spot seems to be at 10000


<!-- .slide: data-state="normal" id="p2m-issues-mem1" data-timing="20s" -->
# Memory usage
<img class="full-slide" src="images/memory-vs-chunksize.svg">


<!-- .slide: data-state="normal" id="p2m-issues-mem2" data-timing="20s" -->
# Runtime
<img class="full-slide" src="images/runtime-vs-chunksize.svg">


<!-- .slide: data-state="normal" id="p2m-issues-ceilometer" data-timing="20s" -->
# Type incompatibilities in Ceilometer

```
DataError: (pymysql.err.DataError)
           (1265, u"Data truncated for column 'timestamp' at row 1")
```

PostgreSQL:<!-- .element: class="fragment" data-fragment-index="1"-->
```
CREATE TABLE public.sample (
  "timestamp" timestamp without time zone,
  recorded_at timestamp without time zone,
  ...);
```
<!-- .element: class="fragment" data-fragment-index="1"-->

MySQL<!-- .element: class="fragment" data-fragment-index="1"-->

```
CREATE TABLE `sample` (
  `timestamp` decimal(20,6) DEFAULT NULL,
  `recorded_at` decimal(20,6) DEFAULT NULL,
  ...);
```
<!-- .element: class="fragment" data-fragment-index="1"-->

Note:
* Ceilometer needs high precision timestamps
* Older MySQL releases don't support that
* Ceilometer is working around that by implementing a TypeDecorator for
  DateTime type and mapping that to a DECIMAL in MySQL
* For PostgreSQL it's just using the available high precision timestamps
* psql2mysql needs to use the same approach (luckily we're using python and
  SQLalchemy, so we could just "borrow" the code)
* During the migration the tool compares the source and target types of the
  table columns, when source == "timestamp" and target == "decimal" the
  TypeDecorator is installed and values will be converted on the fly


<!-- .slide: data-state="section-break" id="p2m-demo" data-timing="20s" -->
# Demo Time


<!-- .slide: data-state="normal" id="p2m-demo-precheck" data-timing="20s" -->
# Demo: Precheck Failure

<asciinema-player id="player" cols="80" rows="16" speed="3" font-size="big" theme="solarized-dark" src="images/precheck-failure.cast"></asciinema-player>



<!-- .slide: data-state="normal" id="p2m-demo-batch" data-timing="20s" -->
# Demo: Batching

<asciinema-player id="player" cols="80" rows="16" speed="3" font-size="big" theme="solarized-dark" src="images/batch.cast"></asciinema-player>


<!-- .slide: data-state="normal" id="p2m-devel" data-timing="20s" -->
# Converting into a module

Note:
* A proof of concept script is great, but doesn't scale.
* Namespaces are great, allows us to logically seperate parts of the code
* In fact, we did this for the Ceilometer high precision timestamp code
* Easier testing, does not require hacks like using importlib


<!-- .slide: data-state="normal" id="p2m-devel" data-timing="20s" -->
# Releasing onto pypi

<img class="full-slide" src="images/pypi-release.jpg" />

Note:
* Being open source is more than just pushing the code up to GitHub, and declaring the job is done.
* To that end, we should make regular releases, and if we push it up to pypi, then we can easily install via pip
* As a bonus, this gives us viewable metrics about installations that GitHub can't readily give
* However, pypi has opinions. Strong ones. The license classifier is only checked on upload, and if it's incorrect, the upload is rejected.


<!-- .slide: data-state="normal" id="p2m-devel" data-timing="20s" -->
# Fun and games with pbr

<img class="full-slide" src="images/pypi-0.4.0-release.jpg" />

Note:
* pbr is Python Build Reasonableness. It gives you a bunch of useful patterns around installation, it's heavily used by OpenStack and sadly no many other projects, but I like it, so let's make use of it.
* pbr is even more opinionated!
* If you choose to use requirements files for dependencies and then use setuptools, there is no simple way to not duplicate them between requirements.txt and setup.py
* Using setup.py install will not install dependencies for pbr-using projects, so we must do it by hand, or use pip
* Including a version number in setup.cfg causes pbr to behave oddly.
* If a long-description isn't specified in setup.cfg, pbr evaluates it to False, and then sets that as the long description anyway


<!-- .slide: data-state="normal" id="p2m-improve" data-timing="20s" -->
# Possible Improvements

Note:
* parallelization
* UI improvments (progress reporting)
* More testing/better testsuite, because testsuites are never ever finished
