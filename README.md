# docker-swarm-cluster

Combines some tooling for creating a good Docker Swarm Cluster.

## Overview

### HTTP(S) Ingress

* Caddy

### Cluster Management

* Swarm Dashboard
* Portainer
* Docker Janitor

### Metrics Monitoring

* Prometheus
* Unsee Alert Manager
* Grafana with some pre-configured made dashboards
* Heavily inspired on https://github.com/stefanprodan/swarmprom

## Installation

* Install Ubuntu on all VMs you're mean't to use in your Swarm Cluster
  * See #Cloud provider tips section for details

* Install the latest Docker package on all VMs (https://docs.docker.com/engine/install/ubuntu/)

* The best practise is to have
  * 3 VMs as Swarm Managers (may be small VMs)
  * any number of VMs as Swarm workers (larger VMs)
  * Place only essential services to run on managers
    * By doing this, in case your services exhaust the cluster resources, you will still have access to portainer and grafana to react to a crisis
    * Avoid your services to run on those machines by using placement constraints:
    * Verify that firewall is either disabled for those internal hosts, or have the correct open ports for mesh service and internal docker overlay network requirements (https://docs.docker.com/network/overlay/#publish-ports-on-an-overlay-network). Those problems are hard to identify, mainly when only ONE VM is with this kind of problem

## Ingress

* Use Caddy to handle TLS (with Let's Encrypt) and load balancing
  * **Indicated for most applications**
  * Just point your DNS entries to the public IP of the VMs that are part of the cluster and they will handle requests and balance between container instances.

* Use a cloud LB to handle front TLS certificates and load balancing
  * **Indicated for heavy loaded or critical sites**
  * Your cloud provider LB will handle TLS certificates and balance between Swarm Nodes. Each Node will have Caddy listening on port 80 through Swarm mesh, so that when a request arrives on HTTP, it will proxy the request to the correct container services based on Host header(according to configured labels)
  * Disable https support from Caddy in this case by using the following label so that it won't be trying to generate a certificate by itself

```yml
  caddy-server:
    deploy:
      labels:
        - caddy.auto_https=off
        - caddy_controlled_server=
    ...
```

```yml
yourservice:
  ...
  deploy:
    placement:
        constraints:
          - node.role != manager
```

* On one of the VMs:
  * Execute ```docker swarm init``` on the first VM with role manager
    * If your machine is connected to more than one network, it may ask you to use `--advertise-addr` to indicate which network to use for swarm communications
  * Copy the provided command/token to run on worker machines (not managers)
  * Execute `docker swarm token-info manager` and keep to run on manager machines

* On machines selected to be managers (min 3)
  * Run the command from previous step for managers and add `--advertise-addr [localip]` with a local IP that connects those machines if they are local so that you don't use a public IP for that (by using Internet link)
    * Ex.: `docker swarm join --advertise-addr 10.120.0.5 --token ...`

* On machines selected to be workers
  * Run the command got on any manager by `docker swarm token-info worker` and add `--advertise-addr [localip]` with a local IP that connects those machines if they are local so that you don't use a public IP for that (by using Internet link)
    * Ex.: `docker swarm join --advertise-addr 10.120.0.5 --token ...`

* Make Docker daemon configurations on all machines
  * **This has to be made after joining Swarm so that network 172.18/24 already exists (!)**
  * Use journald for logging on all VMs (defaults to max usage of 10% of disk)
  * Enable native Docker Prometheus Exporter
  * Unleash ulimit for mem lock (fix problems with Caddy) and stack size
  * Run the following on each machine (workers and managers)

```sh
echo '{"log-driver": "journald", "metrics-addr" : "172.18.0.1:9323", "experimental" : true, "default-ulimits": { "memlock": { "Name": "memlock", "Hard": -1, "Soft": -1 }, "stack": { "Name": "stack", "Hard": -1, "Soft": -1 }} }' > /etc/docker/daemon.json
service docker restart
```

* Start basic cluster services

  * ```git clone https://github.com/flaviostutz/docker-swarm-cluster.git```
  * Take a look at docker-compose-* files for understanding the cluster topology
  * Setup .env parameters
  * Run ```create.sh```

* On one of the VMs, run `curl -kLv --user whoami:whoami123 localhost` and verify if the request was successful

## Security

* Protect all your VMs with a SSH key (https://www.cyberciti.biz/faq/ubuntu-18-04-setup-ssh-public-key-authentication/)
  * If you leave then with weak passwords it's a matter of hours for your server to be hacked (ransomwares mainly)
* Disable access to all ports of your server (but :80 and :443) by configuring your provider's firewall (or by using an internal firewall like iptables)

## Optimal elastic topology

If you need elasticity (need to grow or shrink server size depending on app traffic) a good topology would be to have some two cluster "sizes". One that we call "idle" that has the minimal sizing when few users are on, and a "hot" configuration when traffic is high.

For the "idle" state, we use:

* 1 VM with 1vCPU 2GB RAM (Swarm Manager + Prometheus)
* 2 VMs with 1vCPU 1GB RAM (Swarm Manager)
* 1 VM as worker with 2vCPU 4GB RAM (App services)

For the "hot" state, we use:

* 1 VM with 1vCPU 2GB RAM (Swarm Manager + Prometheus) - same as "idle"
* 2 VMs with 1vCPU 1GB RAM (Swarm Manager) - same as "idle"
* Any number of VMs for handling users load

## HA practices

* Use "spread" preference in your service so that replicas are placed on different Nodes
  * In this example, group spread groups by role manager/worker, but you can group by any other label values

```yml
...
      placement:
        preferences:
          - spread: node.role
...
```

## Service URLs

Services will be accessible by URLs:
    http://portainer.mycluster.org
    http://dashboard.mycluster.org
    http://grafana.mycluster.org
    http://unsee.mycluster.org
    http://alertmanager.mycluster.org
    http://prometheus.mycluster.org

Services which don't have embedded user name protection will use Caddy's basic auth. Change password accordingly. Defaults to admin/admin123admin123

The following services will have published ports on hosts so that you can use swarm network mesh to access admin service directly when Caddy is not accessible
  
* portainer:8181
* grafana: 9191

So point your browser to any public IP of a member VM to this port and access the service

## Common Operations

### Force service rebalancing among nodes

```sh
# docker service ls -q > dkr_svcs && for i in `cat dkr_svcs`; do docker service update "$i" --detach=false --force ; done
for service in $(docker service ls -q); do docker service update --force $service; done
```

WARNING: User service disruption will happen while doing this as some containers will be stopped during this operation

### Add a new VM to the cluster

* Create the new VM on cloud provider on the same VPC (see Cloud provider tips for specific instructions)
* SSH a Swarm manager node and execute `docker swarm join-token worker` to get a Swarm join token
* Copy the command and execute it on new VM
  * Add `--advertise-addr [local-network-interface-ip]` to the command if your host has multiple NICs
  * Execute the command on worker VM. Ex.: `docker swarm join --token aaaaaaaaaaaa 10.120.0.2:2377 --advertise-addr 10.120.0.1`
* All containers that are "global" will be placed on this Node immediatelly
* Even if other hosts are full (containers using too much memory/CPU) they won't be rebalanced as soon this node is added to the cluster. New containers will be placed on this node only when they are restarted (this is by design to minimize user disruption)
* Add the newly created VM to the HTTP Load Balancer (if you use one from cloud provider) so that incoming requests that Caddy will handle will be routed through Swarm mesh network
* Check firewall configuration (either disabled, or configured properly with service mesh and internal overlay network requirements as in https://docs.docker.com/network/overlay/#publish-ports-on-an-overlay-network)

## Production tips

### Optimal Topology

* Have a small VM in your Swarm Cluster to have only basic cluster services. Avoid any other services to run in this server so that if your cluster run out of resources you will still have access to monitoring and admin tools (grafana, portainer etc) so that you can diagnosis what is going on and decide on cluster expansion, for example.

PLACE IMAGE HERE

### OOM

* If a node suffers from severe resource exhaustion, docker daemon presents some strange behavior (services not scheduled well, some commands fail saying the node is not part of a swarm cluster etc). It's better to reboot this VMs after solving the causes.

## Tricks

* Caddy has a "development" mode where it uses a self signed certificate while not in production. Just add `- caddy.tls=internal` label to your service.


## Customizations

1. Change the desired compose file for specific cluster configurations
2. Run ```create.sh``` for updating modified services

## docker-compose files

* Swarm stack doesn't support .env automatically (yet). You have to run ```export $(cat .env) && docker stack...``` so that those parameters work
* docker-compose-ingress.yml
  * ```export $(cat .env) && docker stack deploy --compose-file docker-compose-ingress.yml ingress```
  * Traefik Dashboard: [http://traefik.mycluster.org:6060]()
* docker-compose-admin.yml
  * ```export $(cat .env) && docker stack deploy --compose-file docker-compose-admin.yml admin```
  * Swarm Dashboard: [http://swarm-dasboard.mycluster.org]()
  * Portainer: [http://portainer.mycluster.org]()
  * Janitor: will perform system prune from time to time to release unused resources
* docker-compose-metrics.yml
  * ```export $(cat .env) && docker stack deploy --compose-file docker-compose-metrics.yml metrics```
  * Prometheus: [http://prometheus.mycluster.org]()
  * Grafana: [http://grafana.mycluster.org]()
  * Unsee: [http://unsee.mycluster.org]()
* docker-compose-devtools.yml
  * ```export $(cat .env) && docker stack deploy --compose-file docker-compose-devtools.yml devtools```

### TODO

#### Volume management

* AWS/DigitalOcean BS

#### Logs aggregation

* FluentBit
* Kafka
* Graylog

#### Metrics Monitoring

* Telegrambot

## Cloud provider tips

### Digital Ocean

* For HTTPS certificates, use Let's Encrypt in Load Balancers if you are using a first level domain (something like stutz.com.br). We couldn't manage to make it work with subdomains (like poc.stutz.com.br).

* For subdomains, use certbot and create a wildcard certificate (ex.: *.poc.stutz.com.br) manually and then upload it to Digital Ocean's Load Balancer.

```sh
apt-get install letsencrypt
certbot certonly --manual --preferred-challenges=dns --email=me@me.com --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.poc.me.com
```

#### VMs

* Use image Marketplace -> Docker
* Check "Monitoring" to have native basic VM monitoring from DO panel

