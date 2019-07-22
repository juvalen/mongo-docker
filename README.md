# MongoDB with docker containers in Google Cloud

Creates a mongodb cluster based on docker images in Google Cloud, which are created with Terraform.

The virtual boxes can run docker containers to hold the nodes. Configuration of docker swarm is made with ansible.

MongoDB cluster boxes is installed using docker swarm.

## Terraform setup

This creates three f1-micro instances in Google Cloud.

After creation of 3 instances of Ubuntu 16.04 minimal in Google Cloud from desktop machine (verdi), write down IPs:

0. verdi:  92.58.155.73 (local)

1. docker1: 35.231.170.2 (Google Cloud)

2. docker2: 35.190.177.213 (Google Cloud)

3. docker3: 34.74.47.211 (Google Cloud)

In `hosts` write their external address:

```
[masters]
35.231.170.2

[workers]
35.190.177.213 
34.74.47.211
```

For accessing the, remember that in Google Cloud you first generate the cloud RSA access keys and send public to cloud manager. Then publish in each node the internal address, appending this to /etc/hosts. Ansible will do it as they are hardcoded in the playbook:

```
10.142.0.5 docker1
10.142.0.6 docker2
10.142.0.7 docker3
```

And export the external to <host>:

```
export docker3=35.231.170.2
export docker2=35.190.177.213
export docker1=35.237.123.104
```

And don't forget to open port **2377** for swarm and **27017** for mongo.

In host generate ssl file:

```
openssl rand -base64 741 > mongodb-keyfile
```

`mongodb-keyfile` will be copied from host to `/home/core` in each node, and changed ownerwhip and privileges.

## Install Docker swarm manually

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

See logs from VM with:

`docker logs -ft mongo`

## Install Docker swarm using ansible

Write external addresses of master and workers in `./hosts` file:

```
[masters]
35.237.123.104

[workers]
35.231.170.2
35.190.177.213
```

Chang that public addresses accordingly in hosts and private addresses in `playbook.yml`.

`ansible-playbook -i hosts playnook.yml`

## Deploy mongo swarm

Upload `mongo.compose.yml`to cluster master **docker1**.

Execute:

```
$ docker stack deploy -c mongo-compose.yml mongos
```

## Mongo setup

Connect to `docker1` master isntance and log to the container and add an admin & root user at mongo prompt:

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

## TODO

Do this with:

1. Terraforrm over GCE <DONE

1. Ansible to install a swarm <DONE

1. Deploy mongo images with docker swarm <DONE

## Author

Based in `https://medium.com/@ManagedKube/deploy-a-mongodb-cluster-in-steps-9-using-docker-49205e231319#.mle6a8wmg`

Then setting up a swarm with ansible `https://www.labouardy.com/setup-docker-swarm-on-aws-using-ansible-terraform/`

MongoDB cluster by https://medium.com/@oclausen/deploying-a-mongodb-replica-set-on-aws-docker-swarm-c1c785b61aa9

Data for Google Cloud provider https://www.terraform.io/docs/providers/google/index.html

* **Juan Valentín-Pastrana** (jvalentinpastrana at gmail)

Send feedback if you wish.

