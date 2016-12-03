# Vagrant VirtualBox CoreOS Consul Registrator & Consul-Template

This repo contains a 3 node CoreOS cluster; to demonstrate microservice registration and discovery; using Consul. This repo sets up a nginx loadbalancer.

## Contact
IRC: //TODO - add slack channel info here

Mailing list: // TODO - add mailing list info here

## Streamlined setup

**1)** Install dependencies

* [VirtualBox][virtualbox] 4.3.10 or greater.
* [Vagrant][vagrant] 1.6.3 or greater.

**2)** Clone this project and get it running!

```
git clone https://github.com/stakater/vagrant-coreos-consul-registrator.git
cd vagrant-coreos-consul-registrator
```

**3)** Startup and SSH

There are two "providers" for Vagrant with slightly different instructions.
Follow one of the following two options:

**VirtualBox Provider**

The VirtualBox provider is the default Vagrant provider. Use this if you are unsure.

```
vagrant up
vagrant ssh
```

**VMware Provider**

The VMware provider is a commercial addon from Hashicorp that offers better stability and speed.
If you use this provider follow these instructions.

VMware Fusion:
```
vagrant up --provider vmware_fusion
vagrant ssh
```

VMware Workstation:
```
vagrant up --provider vmware_workstation
vagrant ssh
```

``vagrant up`` triggers vagrant to download the CoreOS image (if necessary) and (re)launch the instance

``vagrant ssh`` connects you to the virtual machine.

Configuration is stored in the directory so you can always return to this machine by executing vagrant ssh from the directory where the Vagrantfile was located.

**4)** Open three terminals and get ssh into each node

**Node 1**

```
vagrant ssh core-01 -- -A
```

**Node 2**

```
vagrant ssh core-02 -- -A
```

**Node 3**

```
vagrant ssh core-03 -- -A
```

You can ignore step 2 & 3

#### Get ip configs to list all network interfaces

```
ipconfig
```

#### Get specific network ip

```
ifconfig eth1 | grep 'inet ' | awk '{ print $2 }'
```

or

```
ifconfig | grep '172.17.8' | awk '{ print $2 }'
```

**5)** Run consul and get them join the cluster

Beware the ip's are hardcoded and they will be same unless someone modifies the `Vagrantfile`

**Node 1**

```
$(docker run --rm progrium/consul cmd:run 172.17.8.101 -d -v /mnt:/data)
```

**Node 2**

```
$(docker run --rm progrium/consul cmd:run 172.17.8.102:172.17.8.101 -d -v /mnt:/data) 
```

**Node 3**

```
$(docker run --rm progrium/consul cmd:run 172.17.8.103:172.17.8.101 -d -v /mnt:/data) 
```

**6)** Export HOST_IP on all 3 nodes

```
export HOST_IP=$(ifconfig eth1 | grep 'inet ' | awk '{ print $2 }')
```

**7)** Run docker registrator on all nodes

```
docker run -d -v /var/run/docker.sock:/tmp/docker.sock --name registrator -h registrator gliderlabs/registrator:latest consul://$HOST_IP:8500
```

**8)** Since now all three nodes are member of same cluster you can run following by sitting on any node; try to change the ip and you will see that the result is same

```
curl 172.17.8.101:8500/v1/catalog/services
```

**9)** Run the sample app on all nodes;

```
docker run -d -P --name node1 -h node1 jlordiales/python-micro-service:latest
docker run -d -P --name node2 -h node2 jlordiales/python-micro-service:latest
docker run -d -P --name node3 -h node3 jlordiales/python-micro-service:latest
```

and then query following and look at the ouput

```
curl 172.17.8.101:8500/v1/catalog/service/python-micro-service
```

and then try to curl the individaul app

```
curl 172.17.8.101:8080/python-micro-service/
```

**10)** Now we will run a frontend load balancer using nginx which will round robin the requests 

while ssh'ed into one of the nodes define a template file with name e.g. `service.ctmpl`

```
upstream python-service {
  least_conn;
  {{range service "python-micro-service"}}server {{.Address}}:{{.Port}} max_fails=3 fail_timeout=60 weight=1;
  {{else}}server 127.0.0.1:65535; # force a 502{{end}}
}

server {
  listen 80 default_server;

  charset utf-8;

  location ~ ^/python-micro-service/(.*)$ {
    proxy_pass http://python-service/$1$is_args$args;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

and then run following nginx container from same node

please look the volume mapping closely!

```
docker run -p 8080:80 -d --name nginx --volume /home/core/templates:/templates --link consul:consul stakater/nginx-with-consul-template:latest

docker exec -it <container-id> /bin/bash
```

_ISSUE_ - currently the nginx container doesn't start by default :( but if I login into the container and run the same command manually it works perfectly fine!

from any of the nodes run this:

```
curl 172.17.8.101:8080/python-micro-service/

while true; do curl 172.17.8.101:8080/python-micro-service/; echo -----; sleep 1; done;
```

#### Shared Folder Setup

There is optional shared folder setup.
You can try it out by adding a section to your Vagrantfile like this.

```
config.vm.network "private_network", ip: "172.17.8.150"
config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true,  :mount_options   => ['nolock,vers=3,udp']
```

After a 'vagrant reload' you will be prompted for your local machine password.

#### Provisioning with user-data

The Vagrantfile will provision your CoreOS VM(s) with [coreos-cloudinit][coreos-cloudinit] if a `user-data` file is found in the project directory.
coreos-cloudinit simplifies the provisioning process through the use of a script or cloud-config document.

Check out the [coreos-cloudinit documentation][coreos-cloudinit] to learn about the available features.

[coreos-cloudinit]: https://github.com/coreos/coreos-cloudinit

#### Configuration

The Vagrantfile will parse a `config.rb` file containing a set of options used to configure your CoreOS cluster.

## Cluster Setup

Launching a CoreOS cluster on Vagrant is as simple as configuring `$num_instances` in a `config.rb` file to 3 (or more!) and running `vagrant up`.
Make sure you provide a fresh discovery URL in your `user-data` if you wish to bootstrap etcd in your cluster.

## New Box Versions

CoreOS is a rolling release distribution and versions that are out of date will automatically update.
If you want to start from the most up to date version you will need to make sure that you have the latest box file of CoreOS. You can do this by running

```
vagrant box update
```

## Docker Forwarding

By setting the `$expose_docker_tcp` configuration value you can forward a local TCP port to docker on
each CoreOS machine that you launch. The first machine will be available on the port that you specify
and each additional machine will increment the port by 1.

Follow the [Enable Remote API instructions][coreos-enabling-port-forwarding] to get the CoreOS VM setup to work with port forwarding.

[coreos-enabling-port-forwarding]: https://coreos.com/docs/launching-containers/building/customizing-docker/#enable-the-remote-api-on-a-new-socket

Then you can then use the `docker` command from your local shell by setting `DOCKER_HOST`:

    export DOCKER_HOST=tcp://localhost:2375

## To Read More!

* Get started [using CoreOS][using-coreos]
* [CoreOS-on-Vagrant](https://coreos.com/os/docs/latest/booting-on-vagrant.html)

[virtualbox]: https://www.virtualbox.org/
[vagrant]: https://www.vagrantup.com/downloads.html
[using-coreos]: http://coreos.com/docs/using-coreos/

## TODO

* Fix the startup of ngnix load balancer - it doesn't startup by default; something is broken! samething works perfect when run from within the container
* Write shell scripts which boostraps everything and one doesn't have to write commands manually
* Write our own sample app