https://coreos.com/os/docs/latest/booting-on-vagrant.html

0) Step

1) Open three terminals and get into each node

```
vagrant ssh core-01 -- -A
```

```
vagrant ssh core-02 -- -A
```

```
vagrant ssh core-03 -- -A
```

You can ignore step 2 & 3

2) Get ip configs to list all network interfaces

```
ipconfig
```

3) Get specific network ip

```
ifconfig eth1 | grep 'inet ' | awk '{ print $2 }'
```

or

```
ifconfig | grep '172.17.8' | awk '{ print $2 }'
```

4) Run consul and get them join the cluster

Beware the ip's are hardcoded and they will be same unless someone modifies the Vagrantfile

```
core-01
$(docker run --rm progrium/consul cmd:run 172.17.8.101 -d -v /mnt:/data)
```

```
core-02
$(docker run --rm progrium/consul cmd:run 172.17.8.102:172.17.8.101 -d -v /mnt:/data) 
```

```
core-03
$(docker run --rm progrium/consul cmd:run 172.17.8.103:172.17.8.101 -d -v /mnt:/data) 
```

5) Export HOST_IP on all nodes

```
export HOST_IP=$(ifconfig eth1 | grep 'inet ' | awk '{ print $2 }')
```

6) Run docker registrator on all nodes

```
docker run -d -v /var/run/docker.sock:/tmp/docker.sock --name registrator -h registrator gliderlabs/registrator:latest consul://$HOST_IP:8500
```

7) Since now all three nodes are member of same cluster you can run following by sitting on any node; try to change the ip and you will see that the result is same

```
curl 172.17.8.101:8500/v1/catalog/services
```

8) Run the sample app on all nodes;

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

9) Now we will run a frontend load balancer using nginx which will round robin the requests 

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
```

_ISSUE_ - currently the nginx container doesn't start by default :( but if I login into the container and run the same command manually it works perfectly fine!

from any of the nodes run this:

```
curl 172.17.8.101:8080/python-micro-service/

while true; do curl 172.17.8.101:8080/python-micro-service/; echo -----; sleep 1; done;
```

backup ...

docker run -p 8080:80 -d --name nginx --volume /home/core/templates:/templates --link consul:consul jlordiales/nginx-consul
