# Metior

This repository consists of a number of configuration files as well as
step-by-step instructions to getting our stack running on Ubuntu.

In the future, there will likely be a script that automates the setup
process.

# Dependencies

## Primary

* [Graphite](http://graphite.wikidot.com)
* [Grafana](https://github.com/grafana/grafana)

## Ancillary

* [Nginx](http://nginx.org)
* [uWSGI](http://projects.unbit.it/uwsgi)
* [elasticsearch](http://elasticsearch.org)

# Setup

## The Virtual Machine

We currently run our data visualization stack on 64-bit Ubuntu 14.04 LTS on
[Parallels](http://www.parallels.com/). You can grab the latest copy of Ubuntu
[here](http://www.ubuntu.com/download/desktop/). We are currently giving the VM
4GB of memory and 4 cores.

## Graphite

The next thing we will wrangle with is getting Graphite installed on the
system. Although it is best practice to install python modules under a
virtual environment, Graphite expects the default installation directory to be
in `/opt/graphite`, which is conveniently behind the home directory and where
most virtual environment setups believe you are installing things. So, for sake
of simplicity, and the fact that we are installing most of the stack in an
isolated environment (on a virtual machine), we will install everything globally.
However, you are free to be a "purist" and attempt to get Graphite working in
a location that lives at the home directory and up (or the other way around and
get the virtual environment to account for modules installed at `/opt/graphite`).
Many of these instructions were borrowed from this
[gist from relaxdiego](https://gist.github.com/relaxdiego/7539911). Deviate from
these instructions at your own risk.


1. `sudo apt-get install python-pip`
2. `sudo pip install Django==1.4.5`
3. `sudo pip install django-tagging`
4. `sudo apt-get install python-dev` http://www.cyberciti.biz/faq/debian-ubuntu-linux-python-h-file-not-found-error-solution/
5. `sudo pip install Twisted==11.1.0`
6. `sudo pip install carbon`
7. `sudo pip install whisper`
8. `sudo chown -R $(whoami):staff /usr/local/bin/whisper*`
9. `sudo chown -R $(whoami):staff /usr/local/bin/rrd2whisper.py`
10. `sudo pip install graphite-web`
11. `sudo chown -R $(whoami):staff /opt/graphite`
12. Override `carbon.conf` found in `/opt/graphite/conf` with the
`conf/graphite/carbon.conf` file found in this repo.
13. Override `storage-schemas.conf` found in `/opt/graphite/conf` with
`conf/graphite/storage-schemas.conf` found in this repo.
14. Override `local_settings.py` found in `/opt/graphite/webapp/graphite` with
`conf/graphite/local_settings.py` found in this repo.
15. `sudo cp -r /opt/graphite/lib/carbon* /usr/local/lib/python2.7/dist-packages/`
16. `sudo cp /opt/graphite/lib/twisted/plugins/* /usr/local/lib/python2.7/dist-packages/twisted/plugins/`
Points 15 & 16 assume you are running python2.7 and that you are on a Debian
Linux distribution (hence dist-packages instead of site-packages). Please tailor
these commands to your situation. These instructions were taken from
[here](http://amin.bitbucket.org/posts/graphite-mac-homebrew.html).
17. `cd /opt/graphite/webapp/graphite && python manage.py syncdb`
18. From the previous command, it is recommended that you create a superuser.
19. `sudo apt-get install libcairo2-dev`
20. `sudo apt-get install libffi-dev`
https://bitbucket.org/cffi/cffi/issue/38/sudo-pip-install-cffi-fails-under-ubuntu
21. `sudo pip install cairocffi`
http://stackoverflow.com/questions/11491268/install-pycairo-in-virtualenv
22. In `/opt/graphite/webapp/graphite/render/glyph.py` remove the `cairo` import
and add the `import cairocffi as cairo`.
23. Add the following aliases to your shell rc file:
`alias carbon='python /opt/graphite/bin/carbon-cache.py start'`
`alias graphite-web='python /opt/graphite/bin/run-graphite-devel-server.py /opt/graphite'`
24. Run both of those aliases, and direct your browser to `http://localhost:8080`.

If you follow the steps above, you are (mostly) on your way to having Graphite
running on your Ubuntu setup. This following step includes a patch that should be
added for pickling and unpickling timeseries data sent to Graphite, which is a
necessary prerequisite for the data visualizations Hapsis needs to display for
the telemetry system.

As of this writing,
[you will also need to apply a patch](https://github.com/graphite-project/graphite-web/issues/608)
to `/opt/graphite/webapp/graphite/util.py` to enable safe unpickeling of timeseries
data:

```
# before other imports ~ line 33
import sys

...

# under SafeUnpickler class definitions ~ lines 90-96 and lines 116-122
PICKLE_SAFE = {
  'copy_reg': set(['_reconstructor']),
  '__builtin__': set(['object', 'list']),
  'collections': set(['deque']),
  'graphite.render.datalib': set(['TimeSeries']),
  'graphite.intervals': set(['Interval', 'IntervalSet']),
}
```

## Getting the First Launch Data

The first launch data is a special case where we conformed to a logging format
that didn't include absolute time information about the data points since the
first version of our telemetry system was running on an Arduino Micro, which has
no system clock. To get this data into your Graphite backend, ensure Carbon is
running and the Graphite Web App and then follow these steps:


1. `sudo add-apt-repository ppa:chris-lea/node.js`
2. `sudo apt-get update`
3. `sudo apt-get install python-software-properties python g++ make nodejs`
4. `sudo npm install -g coffee-script`
5. `git clone git@github.com:hapsis/First-Launch.git` where desired.
6. `cd First-Launch`
7. `./migrate.sh`

You can ensure the data got into Graphite, if, when you visit the web
interface, you can see the tree path `/Graphite/event/0` with the metrics
under that folder.

## Serving Graphite with Nginx and uWSGI

Because we will need to enable
[CORS](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing) when serving
Graphite so that Grafana may access the Graphite API from another port, we will
serve Graphite from Nginx. This section is inspired by [this gist from
drawks](https://gist.github.com/drawks/1830579).

1. `sudo apt-get install nginx`
2. `sudo service nginx start`
3. Ensure Nginx has started by going to the IP address ouptutted by
`ifconfig eth0 | grep inet | awk '{ print $2 }'`.
4. Nginx should already be set up to run at startup, but run
`update-rc.d nginx defaults` for good measure.
5. Remove any other configuration files in `/etc/nginx/sites-available` as well
as their corresponding symlinks in `/etc/nginx/sites-enabled`.
6. Copy `conf/nginx/graphite` found in this repo to `/etc/nginx/sites-available`.
7. `sudo ln -s /etc/nginx/sites-available/graphite /etc/nginx/sites-enabled/graphite`
8. `sudo pip install uwsgi`
9. Copy `conf/uwsgi/graphite.ini` found in this repo to
`/etc/uwsgi/apps-available`.
10. `sudo ln -s /etc/uwsgi/apps-available/graphite.ini /etc/uwsgi/apps-enabled/graphite.ini`
11. Update the previous alias you made for `graphite-web` to be:
`alias graphite-web='uwsgi --ini /etc/uwsgi/apps-enabled/graphite.ini'`

## Grafana

We use Grafana as a better UI Layer to sit on top of Graphite. It requires
elasticsearch and Nginx (or your choice alternative to software to serve
Grafana).

1. `sudo add-apt-repository ppa:webupd8team/java`
2. `sudo apt-get update`
3. `sudo apt-get install oracle-java7-installer`
4. Download and install the ElasticSearch `.deb` file from
[here](http://www.elasticsearch.org/download/).
5. Copy the configuration files found in `conf/elasticsearch` or this repository
to `/etc/elasticsearch`.
6. `sudo service elasticsearch start`
7. Ensure elasticsearch is running properly by directing your browser to
`http://localhost:9200`. You should seem some JSON.
8. Get the latest version of Grafana [here](http://grafana.org/download/).
9. Unpack the file and keep note of where the `src` directory in this package
lives on your machine (you will need this path for Nginx).
10. Replace `confg.sample.js` in the Grafana `src` directory with the config
found in this repo at `conf/grafana/config.js`.
11. Copy `conf/grafana/metior.json` found in this repo to `app/dashboards` in
the `src` folder of the Grafana repo.
12. Add the Nginx configuration file found in this repo at `conf/nginx/metior` to
`/etc/nginx/sites-available`.
13. `sudo ln -s /etc/nginx/sites-available/metior /etc/nginx/sites-enabled/metior`
14. Replace all instances of <PATH_TO_GRAFANA> in the metior Nginx configuration
file with the path to Grafana on your system.
15. `sudo service nginx restart`
16. Check `localhost` and ensure that Grafana is running.
