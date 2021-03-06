
Data Portal for the South African National Treasury
===================================================

This is the software repository for CKAN used as part of the South African National Treasury Data Portal.

We use CKAN to organise the datasets according to various taxonomies and use the CKAN API to make the data discoverable.

Table of contents
-----------------

- [Set up in production](#set-up-in-production)
  - [Solr](#solr)
  - [Redis](#redis)
  - [Postgres](#postgres)
  - [S3](#s3)
  - [CKAN](#ckan)
  - [HTTP Cache](#http-cache)
- [Set up development environment](#set-up-development-environment)
  - [Clone our app repositories](#clone-our-app-repositories)
  - [Edit configuration for development](#edit-configuration-for-development)
  - [Initialise the database](#initialise-the-database)
  - [Create a sysadmin user](#create-a-sysadmin-user)
  - [Set up local hostnames](#set-up-local-hostnames)
  - [Runtime configuration](#runtime-configuration)
  - [Developing our plugins](#developing-our-plugins)
  - [Maintenance](#maintenance)

Set up in production
------------------------

We run CKAN on the dokku platform. We use dokku's dockerfile deployment method to deploy using the the Dockerfile in this repository. Since there are numerous operating system and python dependencies that ckan relies on, we build an image with these on hub.docker.com using Dockerfile-deps.

The Dockerfile then builds on this. We install CKAN plugins using the Dockerfile, which makes it easier to try different ones and keep all plugin installation in one place. These don't take a lot of time so moving them to Dockerfile-deps isn't as important as flexibilty.

This CKAN installation depends on
 - Postgres - main database ad-hoc tables
 - Solr - search on the site
 - Redis - as a queue for background processes
 - S3 - object (file) storage
 - [CKAN DataPusher](https://github.com/OpenUpSA/ckan-datapusher) - [while limited](https://github.com/ckan/ckan/pull/3911), this might help us quickly access data programmatically.
 - NGINX - caching (when needed)

We set up Solr and Redis on the same server and use a remote Postgres instance.

### Solr

Deploy an instance of [Solr configured for CKAN](https://github.com/OpenUpSA/ckan-solr-dokku)

### Redis

We use the dokku Redis plugin.

Install the plugin according to https://github.com/dokku/dokku-redis#installation

```
dokku redis:create ckan-redis
```

### Postgres

Create the database and credentials

```
create user ckan_default with password 'some good password';
alter role ckan_default with login;
grant ckan_default to superuser;
create database ckan_default with owner ckan_default;
-- create datastore user and db
create user datastore_default with password 'some good password';
create database datastore_default with owner ckan_default;
```

*Remember to set the correct permissions for the datastore database*

### S3

Create a bucket and a programmatic access user, and grant the user full access to the bucket with the following policy

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::treasury-data-portal/*",
                "arn:aws:s3:::treasury-data-portal"
            ]
        }
    ]
}
```

### CKAN

Create the CKAN app in dokku

```
dokku apps:create ckan
```

Get the Redis `Dsn` (connection details) for setting in CKAN environment in the next step with `/0` appended.

```
dokku redis:info ckan-redis
```

Set CKAN environment variables, replacing these examples with actual producation ones

- REDIS_URL: use the Redis _Dsn_
- SOLR_URL: use the alias given for the docker link below
- BEAKER_SESSION_SECRET: this must be a secret long random string. Each time it changes it invalidates any active sessions.
- S3FILESTORE__SIGNATURE_VERSION: use as-is - no idea why the plugin requires this.

```
dokku config:set ckan CKAN_SQLALCHEMY_URL=postgres://ckan_default:password@host/ckan_default \
                      CKAN_REDIS_URL=.../0 \
                      CKAN_INI=/ckan.ini \
                      CKAN_SOLR_URL=http://solr:8983/solr/ckan \
                      CKAN_SITE_URL=http://treasurydata.openup.org.za/ \
                      CKAN___BEAKER__SESSION__SECRET= \
                      CKAN_SMTP_SERVER= \
                      CKAN_SMTP_USER= \
                      CKAN_SMTP_PASSWORD= \
                      CKAN_SMTP_MAIL_FROM=webapps+treasury-portal@openup.org.za \
                      CKAN___CKANEXT__S3FILESTORE__AWS_BUCKET_NAME=treasury-data-portal \
                      CKAN___CKANEXT__S3FILESTORE__AWS_ACCESS_KEY_ID= \
                      CKAN___CKANEXT__S3FILESTORE__AWS_SECRET_ACCESS_KEY= \
                      CKAN___CKANEXT__S3FILESTORE__HOST_NAME=http://s3-eu-west-1.amazonaws.com/treasury-data-portal \
                      CKAN___CKANEXT__S3FILESTORE__REGION_NAME=eu-west-1 \
                      CKAN___CKANEXT__S3FILESTORE__SIGNATURE_VERSION=s3v4 \
                      NEW_RELIC_APP_NAME="Treasury CKAN" \
                      NEW_RELIC_LICENSE_KEY=... \
                      CKAN_DISCOURSE_URL= \
                      CKAN_DISCOURSE_SSO_SECRET=
```

Link CKAN and Redis

```
dokku redis:link ckan-redis ckan
```

Link CKAN and Solr

```
dokku docker-options:add ckan run,deploy --link ckan-solr.web.1:solr
```

Link CKAN and CKAN DataPusher

```
dokku docker-options:add ckan run,deploy --link ckan-datapusher.web.1:ckan-datapusher
```

Create a named docker volume and configure ckan to use the volume just so we can configure an upload path. It _should_ be kept clear by the s3 plugin.


```
docker volume create --name ckan-filestore
dokku docker-options:add ckan run,deploy --volume ckan-filestore:/var/lib/ckan/default
```

We customise the app nginx config to

- Allow large file uploads
- Allow a longer request timeout
- Redirect www to non-www (because peope WILL add www to links they shouldn't)
- Log to a second file showing the hostname used to access the server
- To be prepared for caching when needed.

*This breaks letsencrypt renewal so uncomment these and reload nginx to renew the letsencrypt certificate*

Add the following to the logging part of the `http` block of `/etc/nginx/nginx.conf`:

```
    log_format combined '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent"';

    proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=ckan:30m max_size=250m;
    proxy_temp_path /tmp/nginx_proxy 1 2;
```

Add the following nginx config file (and directory if needed) at `/home/dokku/ckan/nginx.conf.d/ckan.conf`:

```

## Caching

proxy_cache ckan;

# Don't cache or served cached copies when any of these authentication
# cookies or headers are set.
proxy_cache_bypass $cookie_auth_tkt$http_x_ckan_api_key$http_authorization;
proxy_no_cache $cookie_auth_tkt$http_x_ckan_api_key$http_authorization;

proxy_cache_valid 30m;
proxy_cache_key $host$scheme$proxy_host$request_uri;

## Uncomment to debug caching
# add_header X-Proxy-Cache $upstream_cache_status;

# Uncomment the next line to enable caching
# proxy_ignore_headers X-Accel-Expires Expires Cache-Control;

## ---

client_max_body_size 100M;
client_body_timeout 120s;

access_log  /var/log/nginx/ckan-access-extended.log ckan;

if ($host = www.budgetportal.openup.org.za) {
  return 301 $scheme://budgetportal.openup.org.za$request_uri;
}

if ($host = www.treasurydata.openup.org.za) {
  return 301 $scheme://treasurydata.openup.org.za$request_uri;
}

## ---

# The X-Frame-Options header indicates whether a browser should be allowed
# to render a page within a frame or iframe.
add_header X-Frame-Options SAMEORIGIN;

# MIME type sniffing security protection
#	There are very few edge cases where you wouldn't want this enabled.
add_header X-Content-Type-Options nosniff;

# The X-XSS-Protection header is used by Internet Explorer version 8+
# The header instructs IE to enable its inbuilt anti-cross-site scripting filter.
add_header X-XSS-Protection "1; mode=block";
```

Then let nginx load it

```
sudo chown dokku:dokku /home/dokku/ckan/nginx.conf.d/ckan.conf
sudo service nginx reload
```

Add the dokku app remote to your local git clone

```
git remote add dokku dokku@dokku7.code4sa.org:ckan
```

Push the app to the dokku remote

```
git push dokku master
```

Set up database and first sysadmin user.

```
dokku run ckan bash
cd src/ckan
paster db init -c /ckan.ini
paster sysadmin add admin email="webapps@openup.org.za" -c /ckan.ini
```

Start the worker:

```
dokku ps:scale ckan worker=1
```

Setup cron jobs.

```
sudo mkdir /var/log/ckan/
sudo touch /var/log/ckan/cronjobs.log
sudo chown ubuntu:ubuntu /var/log/ckan/cronjobs.log
crontab -e

# hourly, update tracking stats, see http://docs.ckan.org/en/ckan-2.7.0/maintaining/tracking.html#tracking
5 * * * * /usr/bin/dokku --rm run ckan paster --plugin=ckan tracking update 2017-09-01 2>&1 >> /var/log/ckan/cronjobs.log && /usr/bin/dokku --rm run ckan paster --plugin=ckan search-index rebuild -r 2>&1 >> /var/log/ckan/cronjobs.log
```

### HTTP Cache


#### CloudFront

Create a Cache Behaviour

- Path pattern: `/`
- Viewer Protocol Policy: `Redirect HTTP to HTTPS`
- Cache Based on Selected Request Headers: `Whitelist`
  - Add custom `x-ckan-api-key`
  - Add standard `Authorization`
- Object Caching: `Customise` and set all TTLs to something sensible like 1800 (30 minutes)
- Forward Cookies: `Whitelist`
  - add `auth_tkt`
- Query String Forwarding and Caching: `Forward all, cache based on all`
- Compress Objects Automatically: `yes`

To enable, ensure it's above the default. To disable, ensure it's below the default.

To invalidate, create an Invalidation with the relevant path, e.g. /* for everything in the Distribution.

#### Nginx

To enable the nginx cache, uncomment `proxy_ignore_headers` in `/home/dokku/ckan/nginx.conf.d/ckan.conf` and reload `nginx:

```
sudo service nginx reload
```

It is important to exempt any authenticated requests from caching. Authenticated requests can be made by the AUTH_TKT cookie, and the Authorization or X-CKAN-AUTH-Key headers. For this reason, publicly-accessible requests should not use authentication.

http://docs.ckan.org/en/ckan-2.7.0/maintaining/installing/deployment.html#create-the-nginx-config-file

To invalidate: `find /path/to/your/cache -type f -delete`

Set up development environment
----------------------------------

While you can set up CKAN directly on your OS, docker-compose is useful to develop and test the docker/dokku-specific aspects.

For development, it is easiest to use docker-compose to build your development environment.

The default development setup doesn't use `discourse-sso-client` because that requires running vulekamali Datamanager side-by-side for authentication. [See how to enable that](#developing-our-plugins) for development of `discourse-sso-client`.

### Clone our app repositories

Clone this repo and supporting repos:

```
git clone git@github.com:OpenUpSA/treasury-ckan.git
git clone git@github.com:OpenUpSA/ckan-solr-dokku.git
git clone git@github.com:OpenUpSA/ckan-datapusher.git
```

### Edit configuration for development

- Remove certain ckan plugins we don't strictly need in development mode.

Edit `ckan.ini` and for the `plugins` entry, remove:
    - s3filestore
    - discourse-sso-client

### Initialise the database

For development, setting up the database in a postgres container is much easier than
running it on your host machine.

The data is persisted using a docker volume.


```
docker-compose run --rm ckan paster --plugin=ckan db init -c /ckan.ini
```

Set up the database for `ckanext-extractor`

```
docker-compose run --rm ckan paster --plugin=ckanext-extractor init -c /ckan.ini
```

### Create a sysadmin user

Create your first admin user. When prompted to create the user, enter `y` and press enter.

```
docker-compose run --rm ckan paster --plugin=ckan sysadmin add admin email="you@domain.com" name=admin password=admin -c /ckan.ini
```

### Set up local hostnames

Set up the hostnames `ckan` and `accounts` to point to `127.0.0.1` in your `hosts` file. This is needed so that ckan's dependencies can refer to it using the internal docker network hostname, and so that you can then access absolute URLs based on that hostname from outside the docker network (on the host computer).

If you need to work with SSO, run Datamanager with something like the following to let CKAN use it for authentication:

### Runtime configuration

Visit `https://ckan:5000` and login with username `admin` and the password `admin`.

Set the homepage layout and colour scheme

1. Click the hammer icon at the top right, beside the link to admin's profile
2. Select the Config tab
3. Change Style to Green
4. Change Homepage to `Search, introductoray area and stats`

Create an organisation named National Treasury. Ensure the slug is `national-treasury`.

### Developing our plugins

Clone the repositories in the directory above this project

```
git clone git@github.com:OpenUpSA/ckanext-satreasury.git
git clone git@github.com:OpenUpSA/ckanext-discourse-sso-client.git
```

Setup development entry points:

```
cd ckanext-satreasury
python setup.py egg_info
cd ../ckanext-discourse-sso-client
python setup.py egg_info
cd ../treasury-ckan
```

And overlay the `docker-compose.yml` file with `docker-compose.plugins.yml`, e.g.

```
docker-compose -f docker-compose.yml -f docker-compose.plugins.yml up`
```

To use SSO authentication in development, run vulekamali Datamanager on the same machine with configuration to enable SSO (same SSO secret, etc) e.g.

```
DJANGO_SITE_ID=2 HTTP_PROTOCOL=http DISCOURSE_SSO_SECRET=d836444a9e4084d5b224a60c208dce14 CKAN_SSO_URL=http://ckan:5000/user/login EMAIL_HOST=localhost EMAIL_PORT=2525 EMAIL_USE_TLS= CKAN_URL=http://ckan:5000 python manage.py runserver
```

Re-enable the `discourse-sso-client` plugin in `ckan.ini` and restart ckan, e.g. with `docker-compose restart ckan`.

Now the login button on `ckan:5000` will redirect you to authenticate with a user on Datamanager. It's easiest to use username+password authentication with a user created in Datamanager.

After authenticating on Datamanager, your browser will be redirected back to CKAN. That user is not a sysadmin by default. Make them a sysadmin using something like `docker-compose run --rm ckan paster --plugin=ckan sysadmin add datamanageruser` where `datamanageruser` is whatever username automatically got generated for you in CKAN after your first SSO login.

### Maintenance

#### Resetting your development environment

To reset a development environment, you need to remove the docker containers **and the volumes**:

```
docker-compose down
```

List the volumes in your docker service to see the names of your named volumes created by docker-compose. The name is usually the name as in `docker-compose.yml` prefixed by the project directory name, e.g.

```
# docker volume ls
DRIVER              VOLUME NAME
...
local               treasury-ckan_ckan-filestore
local               treasury-ckan_db-data
local               treasury-ckan_solr-data
```

Remove them by name, e.g. `docker volume rm treasury-ckan_db-data`

You can then initialise the containers again as in the instructions above.

#### Rebuilding the search index

You might need to rebuild the search index, e.g. if you newly/re-created the docker volume holding the `ckan` solr core data.

```
docker-compose exec ckan bash
cd src/ckan
paster --plugin=ckan search-index rebuild -c /ckan.ini
```

#### Troubleshooting

- If ckan can't connect to solr after rebuilding ckan-solr, restart ckan - I think it's something to do with docker linking the containers. I think Docker needs to link ckan to the new ckan-solr container which happens on restart.
