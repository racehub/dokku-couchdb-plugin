# CouchDB plugin for Dokku

This plugin allows you to run a [CouchDB](http://couchdb.apache.org/) instance in a container alongside a [Dokku](https://github.com/progrium/dokku) application. The container referenced inside is currently locked down at CouchDB 1.6.0. Tested against Docker 1.2.0.

The plugin uses the latest version of the [racehub/couchdb](https://registry.hub.docker.com/u/racehub/couchdb/) container to run Couch.

## Installation

[Dokku](https://github.com/progrium/dokku) and Docker are prequisites, obviously. Once you've got those set up, run the following on the host machine:

```
cd /var/lib/dokku/plugins
git clone https://github.com/racehub/dokku-couchdb-plugin couchdb
dokku plugins-install
```

On Digital Ocean's Ubuntu 14.04 image, I've run into an issue from time to time where `plugins-install` won't pull the backing `racehub/couchdb` container down into docker. If you see an error about a missing container when you run `dokku couchdb:create`, run

```sh
sudo docker pull racehub/couchdb
```

and try again.

## Commands

```
$ dokku help
     couchdb:create <app>     Create a CouchDB container
     couchdb:delete <app>     Delete specified CouchDB container
     couchdb:info <app>       Display database informations
     couchdb:link <app> <db>  Link an app to a CouchDB database
     couchdb:logs <app>       Display last logs from CouchDB container
```

## Configuring your Application

The CouchDB plugin works by injecting environment variables into your application's container. You can use those environment variables to configure your CouchDB client when your app launches.

The environment variables are:

```sh
COUCH_URL=http://root:<password>@$DB_HOST:$PORT"
COUCH_USER=root
COUCH_PASSWORD=<password>
```

The `COUCH_URL` should be enough to access CouchDB from inside your application's container. The `COUCH_USER` is always "root", while the `COUCH_PASSWORD` is a string generated once when the CouchDB container is created. (The password persists if you restart the container, but not if you run `dokku couchdb:delete`.)

## Simple usage

Create a new DB:

```sh
$ dokku couchdb:create foo            # Server side
$ ssh dokku@server couchdb:create foo # Client side

-----> CouchDB container created: couchdb/foo

       Host: 172.16.0.104
       User: 'root'
       Password: 'RDSBYlUrOYMtndKb'
       Database: 'db'
       Public port: 49187
```

Deploy your app with the same name (client side):

```sh
$ git remote add dokku git@server:foo
$ git push dokku master
Counting objects: 155, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (70/70), done.
Writing objects: 100% (155/155), 22.44 KiB | 0 bytes/s, done.
Total 155 (delta 92), reused 131 (delta 80)
remote: -----> Building foo ...
remote:        Ruby/Rack app detected
remote: -----> Using Ruby version: ruby-2.0.0

... blah blah blah ...

remote: -----> Deploying foo ...
remote:
remote: -----> App foo linked to couchdb/foo database
remote:        COUCH_URL=http://root:RDSBYlUrOYMtndKb@172.16.0.104/foo
remote:
remote: -----> Deploy complete!
remote: -----> Cleaning up ...
remote: -----> Cleanup complete!
remote: =====> Application deployed:
remote:        http://foo.server
```

I THINK the linking happens automatically. If not, you'll need to run `dokku couchdb:link foo foo` to get the necessary environment variables in scope.

## Tips on Persistence

`dokku-couchdb-plugin`'s containers will persist the backing database on container restart. If you run `couchdb:create` multiple times, Docker will expose CouchDB over a different port on the host, but all data will survive the restart.

I don't have great advice on a backup strategy at the moment. At RaceHub, we have a [Cloudant](http://cloudant.com/) instance running and set up replication from the new CouchDB container over to our hosted instance. That way if the host is destroyed for some reason, we can bring the database back up right away.

A better strategy would involve firing up a Docker container with access to the shared data volume used by the [racehub/couchdb](https://registry.hub.docker.com/u/racehub/couchdb/) instance and backing up every X minutes to an S3 account.

## Advanced usage

Deleting databases:

```sh
dokku couchdb:delete app-name
```

Linking an app to a specific database:

```sh
dokku couchdb:link app-name database-name
```

CouchDB logs (per database):

```sh
dokku couchdb:logs foo
```

Database information:

```sh
dokku couchdb:info foo
```
