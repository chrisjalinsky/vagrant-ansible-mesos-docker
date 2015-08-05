Vagrant, Mesos, Docker and Ansible
=====================

This is a lab for experimentation with Docker, Vagrant, Ansible, Mesos, Marathon, Zookeeper and HAProxy. It is a fork of the work by Woorank: https://github.com/Woorank/vagrant-mesos-cluster

A vagrant configuration to set up a cluster of mesos master, slaves and zookeepers through ansible. The provision works this way:

* The playbook is run on every host. There are conditionals in the playbook as well as the inventory listings that determine which host qualifies as mesos-slaves and mesos-masters. (Ultimately based on IP addresses and inventory groupings.)
* Common packages are installed on every system. This is the Ansible common role. For instance, NTP is installed along with PIP and other Python libraries.
* In the common role, a hostname fix script is used to resolve thru the loopback. See more here: http://www.debian.org/doc/manuals/debian-reference/ch05.en.html#_the_hostname_resolution
* Mesos and additional packages are configured on a per host basis, and wholly dependent on the Vagrantfile and inventory/vagrant Ansible inventory file for it's unique setup.


There are important dynamic variables set in the playbook, and defined by the inventory. These are used to dynamically write configuration files for Mesos, Zookeeper, and Marathon. They are:

* zk_endpoint
* mesos_zk
* marathon_host_servers
* marathon_zk

# Marathon
Stickiness is added to the HAproxy configuration of the slave servers. A file, /etc/haproxy-marathon-bridge/marathons is written, and /etc/haproxy/haproxy.cfg is watched for changes between the two files. If there are differences, haproxy.cfg is updated


# Usage

Clone the repository, and run:

```
vagrant up
```

If you want to tear the cluster down to rebuild, just run:
```
vagrant destroy
```

This will provision a mini Mesos cluster with one master, and 2 slaves.  The Mesos master server also contains Zookeeper and the
Marathon framework. The slaves will come with Docker and HAProxy installed. You can scale the cluster up by changing the NUM_MASTERS and NUM_SLAVES variables in the Vagrantfile. Keep in mind, you must also mirror those changes in the ansible inventory/vagrant file for the provisioning to work.

# Network settings

This has only been tested using the same subnet, which you can change as well in the Vagrantfile. The default is a 10.0.10 subnet.


# Adjusting the cluster specifications

There are 2 steps to resizing the cluster.

* Adjust the Vagrant VMs (in the Vagrantfile)
You can change the size of the cluster by altering the Vagrantfile and adding more or removing masters/slaves. See the variables below:

```
# Vagrantfile
NETWORK_SUBNET = "10.0.10"
NUM_MASTERS = 1
NUM_SLAVES = 2
MASTER_IPS_START = 10
SLAVE_IPS_START = 100
MASTER_MEMORY = 1024
SLAVE_MEMORY = 1024
MASTER_CPU = 1
SLAVE_CPU = 1
```

*  Adjust the Ansible inventory file (inventory/vagrant).  
You must match the Vagrantfile VMs IP addresses.



# Deploying Docker containers

After provisioning the servers you can access Marathon here (Keep in mind this ip can change if you have more than 1 master):
http://10.0.10.11:8080/ and the master itself here: http://10.0.10.11:5050/

Submitting a Docker container to run on the cluster is done by making a call to
Marathon's REST API:

First create a file, or navigate to `containers/ubuntu.json`, with the details of the Docker container that you want to run: For example,

```
{
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "libmesos/ubuntu"
    }
  },
  "id": "ubuntu",
  "instances": "1",
  "cpus": "0.5",
  "mem": "128",
  "uris": [],
  "cmd": "while sleep 10; do date -u +%T; done"
}
```

And second, submit this container to Marathon by using curl:

```
curl -X POST -H "Content-Type: application/json" http://10.0.10.11:8080/v2/apps -d@ubuntu.json
```

Additionally, there is an simple_webserver.json example container to use.

```
curl -X POST -H "Content-Type: application/json" http://10.0.10.11:8080/v2/apps -d@simple_webserver.json
```

# Building images using Ansible

Building docker images that run an Ansible playbook is interesting. While Dockerfiles can do the same thing, you gain idempotence with Ansible.
You can build and push to Dockerhub (Assuming you have a dockerhub account and a repository that matches the image name). You'll find several examples of Docker builds in the docker_files/ directory. Remember that these folders are synced to the VMs so you can use the slaves to build docker images and push them up to a dockerhub repository to use in your json configs to create containers. For example, There is a simple_webserver.json file located in the /containers folder that references a Docker image built with this method. This one uses Ansible to provision the docker container.

This is based on the work here: https://github.com/ansible/ansible-docker-base

On slave:

* $ cd /vagrant/docker_files/webserver-simple/
* $ sudo docker build -t webserver_simple .
* $ sudo docker login
* $ sudo docker images #Execute to find the name of the image and branch i.e. webserver_simple:latest
* $ sudo docker tag webserver_simple:latest chrisjalinsky/simple_webserver
* $ sudo docker push chrisjalinsky/simple_webserver

Please help me improve this repository. I am interested in merging Kubernetes to test it's orchestral capabilities and reliability in this environment.
