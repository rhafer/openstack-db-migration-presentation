<!-- .slide: data-state="section-break" id="break-psql2mysql" data-timing="10s" -->
# Let's write our own tool!


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

* Subcommands "prechecks" and "purge-tables"
* Batch mode to migrate multiple databases at once

Note:
* Python was more or less the obvisous choice
  * We could leverage a couple of OpenStack libraries (where it made sense)
  * We could use SQLalchemy, which provides nice Abstractions to access
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
* Extensible (type decorators)
* If needed backend specific SQL can still be used (and we needed that)
* OpenStack is using it


<!-- .slide: data-state="normal" id="p2m-issues-solved" data-timing="20s" -->
# Issues we ran into

Note:
* Galera and large transactions
* High precision timestamp workarounds in Ceilometer
* Type for "soft-deletion" not really consistent across different
  OpenStack projects


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


<!-- .slide: data-state="section-break" id="p2m-demo" data-timing="20s" -->
# Demo Time


<!-- .slide: data-state="normal" id="p2m-improve" data-timing="20s" -->
# Possible Improvements

Note:
* parallelization
* UI improvments (progress reporting)
* More testing/better testsuite, because testsuites are never ever finished
