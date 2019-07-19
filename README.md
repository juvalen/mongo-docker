# MongoDB with docker containers in Google Cloud

Creates a mongodb cluster based on docker images in Google Cloud.

The virtual boxes can run docker containers to hold the nodes. Configuration and providioning is made from the host.

## Network setup

Virtual boxes are created in Google Cloud from desktop machine (verdi):

0. verdi:  92.58.155.73 (local)

1. docker1: 34.74.47.211 (Google Cloud)

2. docker2: 35.231.170.2 (Google Cloud)

3. docker3: 35.237.123.104 (Google Cloud)

Remember you generate the cloud RSA access keys and send public to cloud manager. Then run in each node:

```
export docker1=34.74.47.211
export docker2=35.231.170.2
export docker3=35.237.123.104
```

And don't forget to open por **27017**.

In host generate ssl file:

```
openssl rand -base64 741 > mongodb-keyfile
```

And copy it from host to `/home/core` in each node.

Then change `mongodb-keyfile` in nodes to be owned by **999**:

```
sudo chown 999:999 /home/core/mongodb-keyfile
```

## Docker engine

Install docker in each node:

```
sudo apt update
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt -y update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

Add current user to docker group so as not using sudo:

```
sudo usermod -a -G docker <user>

```
And logout.

In docker1 run unauthenticated docker:

```
$ docker run --name mongo \
-v /home/core/mongo-files/data:/data/db \
-v /home/core:/opt/keyfile \
--hostname="docker1" \
-p 27017:27017 \
-d mongo:latest --smallfiles
```

Then log in the container and add an admin & root user at mongo prompt:

```
> use admin
> db.createUser( {
     user: "siteUserAdmin",
     pwd: "123poi",
     roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
   });
> db.createUser( {
     user: "siteRootAdmin",
     pwd: "123poi",
     roles: [ { role: "root", db: "admin" } ]
   });
```

Now restart mongo container authenticated from the VM, remember to run first:

```
$ docker stop mongo
$ docker rm mongo
```

Start mongo in docker1:

```

$ docker run \
--name mongo \
-v /home/core/mongo-files/data:/data/db \
-v /home/core:/opt/keyfile \
--hostname="node1" \
--add-host node1:${docker1} \
--add-host node2:${docker2} \
--add-host node3:${docker3} \
-p 27017:27017 -d mongo:latest \
--smallfiles \
--keyFile /opt/keyfile/mongodb-keyfile \
--replSet "rs0"
```

And next in node2, node3... start mongo image:

```
$ docker run \
--name mongo \
-v /home/core/mongo-files/data:/data/db \
-v /home/core:/opt/keyfile \
--hostname="node2" \
--add-host node1:${docker1} \
--add-host node2:${docker2} \
--add-host node3:${docker3} \
-p 27017:27017 -d mongo:latest \
--smallfiles \
--keyFile /opt/keyfile/mongodb-keyfile \
--replSet "rs0"
```

Add to cluster, from docker1:

```
> use admin
> db.auth("siteRootAdmin", "123poi");
rs0:PRIMARY> rs.add("docker2")`
rs0:PRIMARY> rs.add("docker3")`
```

Initiate the replica:

```
rs.initiate({
      _id: "rs0",
      version: 1,
      members: [
         { _id: 0, host : "docker1:27017" }
      ]
   }
)
```

Check mongo configuration and status:

`rs0:PRIMARY> rs.conf()`

`rs0:PRIMARY> rs.status()`

In secondary nodes run:

```
rs.slaveOk()
```

See logs from VM with:

`docker logs -ft mongo`

## TODO

Do this with:

1. Terraforrm over AWS

1. Ansible to install a swarm

1. Deploy mongo images with docker swarm

## Author

Based in `https://medium.com/@ManagedKube/deploy-a-mongodb-cluster-in-steps-9-using-docker-49205e231319#.mle6a8wmg`

* **Juan Valentín-Pastrana** (jvalentinpastrana at gmail)

Send feedback if you wish.

