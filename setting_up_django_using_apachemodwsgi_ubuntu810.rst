======================================================
Setting up Django using Apache/mod_wsgi on Ubuntu 8.10
======================================================

This article will cover setting up Django using Apache/mod_wsgi on Ubuntu
8.10. The article is targeted at a production environment, but keep in mind
this is a more generalized environment. You may have different requirements,
but this article should at least provide the stepping stones.

The article will use distribution packages where nesscary. As of 8.10 the
Python packages provided by Ubuntu have reached a stable point, in my opinion
of course.

This article will be broken down into small managable chunks. Here is a quick
overview of what will be covered:

* setting up global Python tools
* setting up the database (PostgreSQL or MySQL)
* creating the application environment
* hooking your application into the webserver

Let's get started.

Setting up global Python tools
==============================

Python 2.5 comes pre-installed on Ubuntu 8.10. We will not have to worry about
futzing around getting it installed, even if it was a single command. However,
we want to work with an isoslated environment for our application. This is
encouraged because it will not give you a headache in the future.

Let's get the right tools for this::

    sudo aptitude install python-setuptools
    sudo easy_install pip
    sudo pip install virtualenv

Yes, I do realize that we just used three installers to install three
packages. One day this is will be easier, but I'd say its easy now. We now
have virtualenv (tool for creating virtual Python environments) and pip (tool
for installing Python packages sanely) installed.

The last dependancy we should care about is PIL. If you are going to be using
a Django project that relies on ImageField, this dependancy is required::

    sudo aptitude install python-imaging

Setting up the database
=======================

This section will only cover PostgreSQL and MySQL. Django has support for
SQLite and Orcale. SQLite will work out of the box (it comes with Python 2.5),
but is not recommended in a production environment. I mentioned Orcale because
we have support (enough said).

PostgreSQL
----------

Let's get the PostgreSQL packages::

    sudo aptitude install postgresql-8.3
    sudo aptitude install python-psycopg2

Setup the user and database we will work with::

    su postgres
    createuser botland
    createdb -E utf8 --owner=botland botland_botland
    echo "ALTER USER botland WITH PASSWORD 'password'" | psql template1

MySQL
-----

Let's get the MySQL packages::

    sudo aptitude install mysql-server-5.0
    sudo aptitude install python-mysqldb

Setup the user and database we will work with::

    mysql --user=root -p

Once in the console type::

    CREATE DATABASE botland_botland;
    CREATE USER 'botland'@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON botland_botland.* TO 'botland'@'localhost';

Creating the application environment
====================================

The first step is create a user where we want to run the application::

    adduser botland

Once your application user is created become that user. Next, we need to setup
our virtual Python environment::

    mkdir ~/virtualenvs
    virtualenv ~/virtualenvs/botland
    source ~/virtualenvs/botland/bin/activate
    easy_install pip

The last command is important because pip will not work inside the virtual
environment unless installed "locally".

You can now install any dependancies that are application specific. These will
be installed in the isolated environment named botland. For purporses of this
article the only dependancy is Django::

    pip install Django

The next steps are going to be very application specific. For this article,
we are going to create a new application. However, in your case, you may
already have one. The important bits is that we are going to store the Django
project in the ``~/webapps`` directory. Keep that in mind as you read the rest
of the article such that you can modify paths according to your application.

Let's create the application we will use for the rest of the article::

    mkdir ~/webapps
    cd ~/webapps
    django-admin.py startproject botland_project

Create a ``local_settings.py`` file in the Django project directory with
settings specific to the server. In this case this will consist of database
settings::

    DATABASE_ENGINE = "postgresql_psycopg2" # change this to "mysql" for MySQL
    DATABASE_NAME = "botland_botland"
    DATABASE_USER = "botland"
    DATABASE_PASSWORD = "password"
    DATABASE_HOST = "127.0.0.1"
    
Hook in the ``local_settings.py`` file in to your ``settings.py`` by adding
the following lines to the bottom::

    try:
        from local_settings import *
    except ImportError:
        pass

Let's sync the database::

    python manage.py syncdb

The last thing we need to do for our application is to create a WSGI file that
Apache's mod_wsgi will need to hook into Django. Create ``botland.wsgi`` in a
directory named ``deploy`` in your project::

    import os
    import sys

    # put the Django project on sys.path
    sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "../../")))

    os.environ["DJANGO_SETTINGS_MODULE"] = "botland_project.settings"

    from django.core.handlers.wsgi import WSGIHandler
    application = WSGIHandler()

Hooking your application into the webserver
===========================================

Let's finish off the article by hooking your application into Apache. Get the
required distribution packages installed::

    sudo aptitude install apache2
    sudo aptitude install libapache2-mod-wsgi

We will create a new virtual host for our application. The virtual host will
be ``example.com`` for this article, but change this to a name that you will
want to use. You could just override the default virtual host if you'd like.

Create a new file, ``/etc/apache2/sites-available/example.com``::

    <VirtualHost *:80>
        ServerName example.com

        WSGIDaemonProcess botland-production user=botland group=botland threads=10 python-path=/home/botland/virtualenvs/botland/lib/python2.5/site-packages
        WSGIProcessGroup botland-production

        WSGIScriptAlias / /home/botland/webapps/botland_project/deploy/botland.wsgi
        <Directory /home/botland/webapps/botland/deploy>
            Order deny,allow
            Allow from all
        </Directory>

        ErrorLog /var/log/apache2/error.log
        LogLevel warn

        CustomLog /var/log/apache2/access.log combined
    </VirtualHost>

Enable the virtual host file we created::

    cd /etc/apache2/sites-enabled
    ln -s ../sites-available/example.com

Restart Apache::
    
    /etc/init.d/apache2 restart

You should now be able to access the application by pointing your web browser
to the virtual host (don't actually try example.com it won't work).