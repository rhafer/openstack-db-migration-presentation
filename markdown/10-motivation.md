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
