<!-- .slide: data-state="normal" id="nested-lists" data-timing="120s" data-menu-title="Motivation" -->
# Motivation

### PostgreSQL is a great DB, why switch away from it?

> "[..] currently in the OpenStack community there is a bias towards MySQL over other
> databases like for example PostgreSQL [..]"
>
> -- <cite>[OpenStack TC Resolutions](https://governance.openstack.org/tc/resolutions/20170613-postgresql-status.html)</cite>

Note:
* PostgreSQL is still supported by OpenStack and works fine in most cases
* there have only been a few PostgreSQL specific issues in some
  OpenStack projects
* but MySQL is getting more attention upstream
  * more testing in the upstream CI
  * more experience upstream
* according to the 2017 OpenStack User Survey less then 10% of the deployments
  are on PostgreSQL
* The support status topic for PostgreSQL was raised in the Technical Commitee
  in 2017. It ended with a resolution that expresses a bias towards MySQL
* So we decided to spend some time to research on possible ways to migrate
  from PostgreSQL to MySQL
* Whenever we mention "MySQL" we're usually not referring to MySQL only but
  all the different flavours that exist of MySQL (MariaDB, Percona, ...)


<!-- .slide: data-state="normal" id="lets-migrate" data-timing="10s" -->
# Let's just migrate.
<img class="full-slide" src="images/how-hard-can-it-be.jpg" />


<!-- .slide: data-state="normal" id="dump-reload" data-timing="10s" -->
# It's all just SQL!

<code style="font-size: 5rem; display: block; margin: 15%">
    pg_dumpall | mysql
</code>

Note:
* Let's just dump all Postgres data to SQL and import it using mysql
* Unfortunately it's not that easy


<!-- .slide: data-state="normal" id="sql-diff" data-timing="90s" -->
# ... or why a simple dump and reload won't work

* Syntax differences
  * Quotes, Comments, case-sensitivity
* Data Types
  * MySQL Text Types have a size limit (TEXT can be 64kB)
  * different accuracy of default Time/Date types
  * BOOLEAN vs TINYINT

Note:
* SQL is not fully standardized
* Just to list a few differences that affect OpenStack to some extend at
  least
* "TEXT" is used for quite a few collumns in OpenStack
  * Mainly "description" and similar columns


<!-- .slide: data-state="normal" id="openstack-diff" data-timing="120s"  -->
# Other differences (default charsets)
### Have you ever tried this:

  `openstack volume create --size 1 "My ` &#x1F4BE;`"`

* It works with PostgreSQL <!-- .element: class="fragment" -->
* But with MySQL:<!-- .element: class="fragment" -->

<asciinema-player id="player" cols="80" rows="6" speed="3" font-size="big" theme="tango" src="images/emoji.cast"></asciinema-player>
<!-- .element: class="fragment emoji-cast" -->

Note:
* Many OpenStack projects still default to the "UTF8" charset when using
  mysql. Especially as of OpenStack Newton.
* UTF8 in MySQL is not real UTF8
  * It's limited to 3 Byte UTF-8 characters
  * that reflects Unicode Codepoint 0x0000 to 0xFFFF (Basic Multilingual
    Plane)
  * It covers most commonly used characters
  * But misses Emojis, Mathematical Symbols and some less often used CJK
    characters
* PostgreSQL supports the full utf8 range by default
* In MySQL for full "UTF8" support the "utf8mb4" charset needs to be used
* There is no good way to solve this issue programmatically
  * trying to convert from "real" utf8 to mysql's 3byte encode would always
  mean data loss
  * Any tool we use for the migration would need to be able to detect issues
    and provide pointers on how to fix them
* Converting every OpenStack project to use "utf8mb4" is also not easy to
  undertake. It has been raised in the past, but never really completed.
  It's also unclear what this would entail for existing deployments.
* Luckily Characters > 0xFFFF are unlikely to appear in OpenStack database but
  we need to be aware of it for the migration


<!-- .slide: data-state="normal" id="openstack-diff-2" data-timing="45s" -->
# Backend specific differences in OpenStack
### some projects use slightly different schemas depending on database backend

Note:
* to workaround limits of older MySQL release ceilometer decided to "emulate"
  high precision timestamps by using "DECIMAL".
* for PostgreSQL is using the available TimeStamp type
