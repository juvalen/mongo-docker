# MongoDB with docker containers (Vagrant/Virtualbox)

https://medium.com/@ManagedKube/deploy-a-mongodb-cluster-in-steps-9-using-docker-49205e231319#.mle6a8wmg

Create and start three nodes in Vagrant, with `boot2docker.iso`:

0. host:  192.168.99.26

Add an extra IP to host with:

`$ sudo ip addr add 192.168.99.26/24 dev wlp16s0p` 

1. node1: 192.168.99.106

2. node2: 192.168.99.105

3. node3: 192.168.99.103

Run in each node:

```
$ export node1=192.168.99.106
$ export node2=192.168.99.105
$ export node3=192.168.99.103
```

In host generate ssl file:

```
$ openssl rand -base64 741 > mongodb-keyfile

$ sudo chown 999:999 /home/core/mongodb-keyfile 
```

And copy it to `/home/core` in each node:

`$ docker-machine scp mongodb-keyfile node1:/home/core`

`$ docker-machine scp mongodb-keyfile node2:/home/core`

`$ docker-machine scp mongodb-keyfile node3:/home/core`

Change `keyfile` in nodes to be owned by **999**.

In node1 run unauthenticated docker:

```
$ docker run --name mongo \
-v /home/core/mongo-files/data:/data/db \
-v /home/core:/opt/keyfile \
--hostname="node1" \
-p 27017:27017 \
-d mongo:latest --smallfiles
```

Then add admin & root user at mongo prompt:

```
> db.createUser( {
     user: "siteUserAdmin",
     pwd: "password",
     roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
   });
```

```
> db.createUser( {
     user: "siteRootAdmin",
     pwd: "password",
     roles: [ { role: "root", db: "admin" } ]
   });
```

Now restart mongo container authenticated, remember to run first:

```
$ docker stop mongo
$ docker rm mongo
```

Start mongo in node1:

```
$ docker run \
--name mongo \
-v /home/core/mongo-files/data:/data/db \
-v /home/core:/opt/keyfile \
--hostname="node1" \
--add-host node1:${node1} \
--add-host node2:${node2} \
-p 27017:27017 -d mongo:latest \
--smallfiles \
--keyFile /opt/keyfile/mongodb-keyfile \
--replSet "rs0"
```

And next in node2 start mongo image:
```
$ docker run \
--name mongo \
-v /home/core/mongo-files/data:/data/db \
-v /home/core:/opt/keyfile \
--hostname="node2" \
--add-host node1:${node1} \
--add-host node2:${node2} \
-p 27017:27017 -d mongo:latest \
--smallfiles \
--keyFile /opt/keyfile/mongodb-keyfile \
--replSet "rs0"
```

Add to cluster, from node1:

`rs0:PRIMARY> rs.add("node2")`

Check configuration and status:

`rs0:PRIMARY> rs.conf()`

See logs with:

`$ docker logs -ft mongo`

`rs0:PRIMARY> rs.status()`

