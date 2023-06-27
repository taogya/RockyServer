# RockyServer
Base Web Server for development on RockyLinux with Django, uWSGI, PostgreSQL


# Overview
example built with UTM on Mac Book.  
This is do task below.
  - [Setup Firewall](#firewall)
  - [Add user for Django](#adduser)
  - [Setup Database](#database)
  - [Setup Git](#git)
  - [Setup Django](#django)
  - [Setup uWSGI](#uwsgi)
   
The administrator user is "admin".  

<a id="firewall"></a>
## Setup Firewall
```sh
$ sudo firewall-cmd --add-port=8000/tcp --zone=public --permanent
$ sudo firewall-cmd --reload
$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s1 enp0s2
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 8000/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```

<a id="adduser"></a>
## Add user for Django
```sh
$ sudo adduser rocky
```

<a id="database"></a>
## Setup Database
```sh
# install package
$ sudo yum install -y postgresql-server

# setup postgresql
$ sudo postgresql-setup --initdb
$ sudo cp /var/lib/pgsql/data/pg_hba.conf /var/lib/pgsql/data/pg_hba.conf.org
$ sudo vi /var/lib/pgsql/data/pg_hba.conf
:
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident -> md5
# IPv6 local connections:
host    all             all             ::1/128                 ident -> md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            ident
host    replication     all             ::1/128                 ident

# start postgresql
$ sudo systemctl enable postgresql.service
$ sudo systemctl start postgresql.service

# (option)drop database
$ sudo su - postgres << EOS
psql -U postgres -t -P format=unaligned -c 'drop database rocky_db;'
psql -U postgres -t -P format=unaligned -c "drop user \"rocky\";"
EOS

# create database
$ sudo su - postgres << EOS
psql -U postgres -t -P format=unaligned -c 'create database rocky_db encoding utf8;'
psql -U postgres -t -P format=unaligned -c "create user \"rocky\" with password 'rocky_pass';"
psql -U postgres -t -P format=unaligned -c 'alter database rocky_db owner to "rocky";'
psql -U postgres -t -P format=unaligned -c 'grant all on database rocky_db to "rocky";'
EOS
```

<a id="git"></a>
## Setup Git
```sh
# install git
$ sudo yum install -y git
$ sudo su - rocky
$ git config --global user.name "your_name"
$ git config --global user.email "your_email"

# setup ssh
$ mkdir .ssh && chmod 700 .ssh
$ ssh-keygen -t rsa -f ~/.ssh/rocky_rsa
-> set ssh public key as .ssh/rocky_rsa.pub in your github
$ echo -n 'Host github.com
  HostName github.com
  User git
  Port 22
  IdentityFile ~/.ssh/rocky_rsa
  PreferredAuthentications publickey
  TCPKeepAlive yes 
  IdentitiesOnly yes' > ~/.ssh/config
$ chmod 600 .ssh/*

# connection test
$ ssh -T git@github.com
Enter passphrase for key '/home/rocky/.ssh/rocky_server_rsa': -> your secret key password
Hi xxxxx! You've successfully authenticated, but GitHub does not provide shell access.

# clone repository
$ mkdir -p app/ && cd app
$ git clone -b develop git@github.com:taogya/RockyServer.git
```

<a id="django"></a>
## Setup Django
```sh
# install package
$ sudo yum install -y gcc python3-devel memcached

# setup python
$ sudo su - rocky
$ cd app/RockyServer
$ mkdir log static
$ python -V
Python 3.9.14
$ python -m venv .venv
$ . .venv/bin/activate
$ pip install isort flake8 autopep8 radon
$ pip install django django-widget-tweaks django-environ django-admin-rangefilter django-debug-toolbar
$ pip install psycopg2-binary pymemcache
$ pip install uwsgi
$ pip install numpy pandas
$ pip freeze | tee requirements.txt
asgiref==3.7.2
autopep8==2.0.2
colorama==0.4.6
Django==4.2.2
django-admin-rangefilter==0.10.0
django-debug-toolbar==4.1.0
django-environ==0.10.0
django-widget-tweaks==1.4.12
flake8==6.0.0
isort==5.12.0
mando==0.7.1
mccabe==0.7.0
numpy==1.25.0
pandas==2.0.2
psycopg2-binary==2.9.6
pycodestyle==2.10.0
pyflakes==3.0.1
pymemcache==4.0.0
python-dateutil==2.8.2
pytz==2023.3
radon==6.0.1
six==1.16.0
sqlparse==0.4.4
tomli==2.0.1
typing_extensions==4.6.3
tzdata==2023.3
uWSGI==2.0.21

# create secret key
$ python -c "from django.core.management.utils import get_random_secret_key;print(get_random_secret_key())"
-> your_secret_key

# setup project
$ django-admin startproject rocky
$ mkdir rocky/static rocky/templates
# create env
$ vi rocky/rocky/.env
# ### Project Configration
# BASE_DIR should be project_dir/app_name.
BASE_DIR=/home/rocky/app/RockyServer/rocky
VENV_DIR=/home/rocky/app/RockyServer/.venv
STATIC_ROOT=/home/rocky/app/RockyServer/static
LOG_DIR=/home/rocky/app/RockyServer/log
# ### Database Configration
DB_ENGINE=django.db.backends.postgresql
DB_NAME=rocky_db
DB_USER=rocky
DB_PASSWORD="rocky_pass"
DB_HOST=localhost
DB_PORT=5432
# ### Application Configration
# auto set if SECRET_KEY is empty.
CACHE_URL=pymemcache://127.0.0.1:11211
SECRET_KEY="your_secret_key"
ALLOWED_HOSTS=localhost
DEBUG=True
ROOT_URLCONF=rocky.urls
WSGI_APPLICATION=rocky.wsgi.application

# modify settings
$ vi rocky/rocky/settings.py

 :
 
import os
from pathlib import Path

import environ
from django.conf.global_settings import DATETIME_INPUT_FORMATS

# Take environment variables from .env file
env = environ.Env(
    # set casting, default value
    DEBUG=(bool, False)
)
environ.Env.read_env(os.path.join(Path(__file__).resolve().parent, '.env'))

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = env('BASE_DIR')

# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/4.2/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env('DEBUG')

ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')


# Application definition

INSTALLED_APPS = [
    :
    'widget_tweaks',
    'rangefilter',
]
PROJECT_APPS = [
]
INSTALLED_APPS += PROJECT_APPS

:

ROOT_URLCONF = env('ROOT_URLCONF')

TEMPLATES = [
    {
        :
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        :
]

WSGI_APPLICATION = env('WSGI_APPLICATION')


# Database
# https://docs.djangoproject.com/en/4.2/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': env('DB_ENGINE'),
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': int(env('DB_PORT')),
    },
}
CACHES = {
    'default': env.cache(),
}

:

# Internationalization
# https://docs.djangoproject.com/en/4.2/topics/i18n/

LANGUAGE_CODE = 'ja'
TIME_ZONE = 'Asia/Tokyo'
USE_I18N = True
USE_TZ = True
DATETIME_FORMAT = 'Y/m/d H:i:s'
USE_L10N = False
DATETIME_INPUT_FORMATS += ('%Y-%m-%d %H:%M:%S.%f%z',)


# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/4.2/howto/static-files/

STATIC_URL = '/static/'
STATIC_ROOT = env('STATIC_ROOT')
STATICFILES_DIRS = (
    os.path.join(BASE_DIR, 'static'),
)

:

# CommonLogger Settings
LOG_DIR = env('LOG_DIR')
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "root": {
        "handlers": ["console"],
        "level": "DEBUG",
        "propagate": False,
    },
    "formatters": {
        "verbose": {
            "format": "[{asctime}][{module}][{process:d}][{thread:d}][{levelname}][{message}]",
            "style": "{",
        },
        "simple": {
            "format": "[{asctime}][{levelname}][{message}]",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "level": "INFO",
            "class": "logging.StreamHandler",
            "formatter": "simple",
        },
        "general": {
            "level": "INFO",
            "class": "logging.handlers.WatchedFileHandler",
            "filename": os.path.join(LOG_DIR, "general.log"),
            "formatter": "simple",
        },
    },
    "loggers": {
        "django": {
            "handlers": ["console"],
            "propagate": False,
        },
        "general": {
            "handlers": ["console"],
            "propagate": False,
        },
    },
}

# Debug Settings
if DEBUG:
    # pip install django-debug-toolbar
    INSTALLED_APPS += [
        'debug_toolbar',
    ]

    MIDDLEWARE += [
        'debug_toolbar.middleware.DebugToolbarMiddleware',
    ]

# Access Test of Django
$ python rocky/manage.py makemigrations
No changes detected
$ python rocky/manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying sessions.0001_initial... OK
$ python rocky/manage.py runserver 0.0.0.0:8000
```
-> access to http://localhost:8000/admin
   If admin view is displayed, it is success.

<a id="uwsgi"></a>
## Setup uWSGI
```sh
$ sudo su - rocky
$ cd app/RockyServer

# collectstatic
$ python rocky/manage.py collectstatic

# create ini
$ mkdir -p conf/uwsgi
$ vi conf/uwsgi/uwsgi.ini
[uwsgi]
# same to VENV_DIR
home = /home/rocky/app/RockyServer/.venv

socket = /var/run/uwsgi/uwsgi.sock
pidfile = /var/run/uwsgi/master.pid
http = 0.0.0.0:8000
master = true
vacuum = true
chmod-socket = 666
uid = rocky
gid = rocky

processes = 2
enable-threads = true
threads = 4

logto = /var/log/uwsgi/uwsgi.log
log-reopen = true
log-x-forwarded-for = true

# same to BASE_DIR
chdir = /home/rocky/app/RockyServer/rocky
wsgi-file = /home/rocky/app/RockyServer/rocky/rocky/wsgi.py
static-map = /static=/home/rocky/app/RockyServer/static
harakiri = 20
max-requests = 5000
buffer-size = 32768

# create service
$ vi conf/uwsgi/uwsgi.service
[Unit]
Description=uWSGI Service
After=syslog.target

[Service]
User=rocky
Group=rocky
WorkingDirectory=/home/rocky/app/RockyServer/
ExecStart=/bin/sh -c '\
    . .venv/bin/activate;\
    uwsgi --ini conf/uwsgi/uwsgi.ini;\
'
RuntimeDirectory=uwsgi
Restart=always
KillSignal=SIGQUIT
StandardError=syslog
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target

# start service
$ sudo cp /home/rocky/app/RockyServer/conf/uwsgi/uwsgi.service /etc/systemd/system/uwsgi.service
$ sudo mkdir -p /var/log/uwsgi
$ sudo chown rocky:rocky /var/log/uwsgi
$ sudo systemctl daemon-reload
$ sudo systemctl start uwsgi
```
-> access to http://localhost:8000/admin  
   If admin view is displayed, it is success.
