graphite-metrics: standalone graphite collectors for various stuff not (or poorly) handled by other monitoring daemons
--------------------

Notable stuff here:


### sa_carbon.py

Script to run once-in-a-while (from 1 min to 1 month) to push data, collected by
[sysstat](http://sebastien.godard.pagesperso-orange.fr/) into graphite.

Uses `sadf -j` json export for /var/log/sa files, collected by sysstat, usually
by sadc (in my case - `sadc -F -L -S DISK -S XDISK -S POWER 60`).


### harvestd.py

Simple daemon, which collects metric values and sends them to graphite once per
interval.

Consists of various components for processing of:

* /proc/slabinfo for useful-to-watch values, not everything.
* /proc/vmstat and /proc/meminfo in a consistent way.
* /proc/stat for irq, softirq, forks.
* /proc/buddyinfo and /proc/pagetypeinfo (memory fragmentation).
* /proc/interrupts and /proc/softirqs.
* Cron log to produce start/finish events and duration for each job into a
	separate metrics, adapts jobs to metric names with regexes.
* Per-system-service accounting using systemd cgroups.

Plan is to split all this accumulated stuff into separate binaries, maybe
feeding data to statsd, not carbon, although I don't see enough benefit there,
which can justify added fail and complexity.
That's what the project is mostly for.


Rationale
--------------------

Most other tools can (in theory) collect this data, and I've used
[collectd](http://collectd.org) for most of these, but it:

* Doesn't provide some of the most useful stuff - nfs stats, disk utilization
	time percentage, etc.

* Fails to collect some other stats, producing bullshit like 0'es,
	clearly-insane or negative values (for io, network, sensors, ...).

* General-purpose plugins like "tail" add lot of complexity, making
	configuration into a mess, while still lacking some basic functionality which
	10 lines of code can easily provide.

* Mangles names for metrics, as provided by /proc and referenced in kernel docs
	and on the internets, no idea what the hell for, "readability"?

Initially I've tried to implement these as collectd plugins, but it's python
plugin turned out to be leaking RAM and collectd itself segfaults something like
once-a-day, even in the latest releases (although probably because of bug in
some plugin).

Plus, collectd data requires post-processing anyway - proper metric namespaces,
counters, etc.

Given that the alternative is to just get the data and echo it as "name val
timestamp" to tcp socket, I just don't see why would I need all the extra
complexity and fail that collectd provides.

Other than collectd, I've experimented with
[ganglia](http://ganglia.sourceforge.net/), but it's static schema is a no-go
and most of stuff there doesn't make sense in graphite context.