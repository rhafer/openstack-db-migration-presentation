<!-- .slide: data-state="section-break" id="break-psql2mysql" data-timing="10s" -->
# Let's write our own tool!


<!-- .slide: data-state="normal" id="p2m-goals" data-timing="20s" -->
# Goals

### KISS principle

Note:
* Open Source
* Newton, but should work with other OpenStack releases
* It's ok to make it somewhat specific to OpenStack databases
* Allow to validate source data before running the actual migration and point
  the user to problematic Rows/Values


<!-- .slide: data-state="normal" id="p2m-intro" data-timing="20s" -->
# Choices

Note:
* At SUSE we tend to have a little bit of bias towards Ruby
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
# SQLAlchemy for the win!

Note:
* Introspection to read Tables and Columns
* Type Abstractions defined for most types used by OpenStack
* Extensible (type decorators)
* If needed backend specific SQL can still be used (and we needed that)
* OpenStack is using it


<!-- .slide: data-state="normal" id="p2m-issues-solved" data-timing="20s" -->
# Issues we ran into

Note:
* Galera and large transactions
* Ceilometer high precision timestamp workarounds in Ceilometer
* Type for "soft-deletion" not really consistent across different
  OpenStack projects


<!-- .slide: data-state="normal" id="p2m-devel" data-timing="20s" -->
# Converting into a module

Note:
* A proof of concept script is great, but doesn't scale.
* Requires hacks like importlib


<!-- .slide: data-state="normal" id="p2m-devel" data-timing="20s" -->
# Releasing onto pypi

Note:
* pypi is ... opinionated.


<!-- .slide: data-state="normal" id="p2m-devel" data-timing="20s" -->
# Fun and games with pbr

Note:
* pbr is even more opinionated!


<!-- .slide: data-state="section-break" id="p2m-demo" data-timing="20s" -->
# Demo Time


<!-- .slide: data-state="normal" id="p2m-improve" data-timing="20s" -->
# Possible Improvements

Note:
* parallelization
* UI improvments (progress reporting)
* More testing/better testsuite, because testsuites are never ever finished