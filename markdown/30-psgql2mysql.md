<!-- .slide: data-state="section-break" id="break-psql2mysql" data-timing="10s" -->
# Let's write our own tool!


<!-- .slide: data-state="normal" id="p2m-goals" data-timing="20s" -->
# Goals

* Open Source
* Newton, but should work with other OpenStack releases
* It's ok to make it somewhat specific to OpenStack databases
* Allow to validate source data before running the actual migration and point
  the user to problematic Rows/Values

### ... keep it simple stupid!

# Choices

* Python (2.7, 3.5 and 3.6)
* SQLalchemy


<!-- .slide: data-state="normal" id="p2m-intro" data-timing="20s" -->
# Introducing psql2mysql

https://github.com/SUSE/psql2mysql

* precheck
* migrate
* purge-tables
* batch


<!-- .slide: data-state="normal" id="p2m-internals" data-timing="20s" -->
# SQLAlchemy for the win

* Introspection to read Tables and Columns
* Type Abstractions defined for most types used by OpenStack
* Extensible (type decorators)
* If needed backend specific SQL can still be used (and we needed that)
* OpenStack is using it


<!-- .slide: data-state="normal" id="p2m-issues-solved" data-timing="20s" -->
# Issues we ran into

* Galera and large transactions
* Ceilometer high precision timestamp workarounds in Ceilometer
* Type for "soft-deletion" not really consistent across different
  OpenStack projects


<!-- .slide: data-state="normal" id="p2m-devel" data-timing="20s" -->
# Converting into a module, releasing onto pypi, fun and games with pbr


<!-- .slide: data-state="section-break" id="p2m-demo" data-timing="20s" -->
# Demo Time


<!-- .slide: data-state="normal" id="p2m-improve" data-timing="20s" -->
# Possible Improvements
* parallelization
* UI improvments (progress reporting)
* More testing/better testsuite, because testsuites are never ever finished
