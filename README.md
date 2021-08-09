# monitoring-stack

Sample monitoring stack (logs + metrics) build based on:

- Prometheus - metrics db
- Grafana - logs and metrics display
- cAdvisor - container metrics
- NodeExporter - docker host metrics
- AlertManager - alerts based on metrics db
- Loki - container logs aggregation
- swag/caddy - reverse proxy + basicauth + TLS

Based on: https://github.com/stefanprodan/dockprom extended with Loki, wrapped with ansible for full server provisioning.

## TODO

[x] Selinux provisioning (based on geerlingguy role), ssh config, security hardening

[x] Firewall configuration (based on geerlingguy role), open ports:

- 22 (SSH)
- 80 (HTTP)
- 443 (HTTPS)

[x] Docker installation (based on geerlingguy role)

[X] 4. Write full docker-compose file based on https://github.com/stefanprodan/dockprom (check documentation for upgrades, and how this should be started) + add Loki to the compose file. Everything should be running inside one docker network.

[x]5. Create ansible role (central_node) which will edit necessary configs and start services from docker-compose required to run the monitoring infrastructure. Required services should be exposed to outside via caddy on TLS on different subdomains. Also a parameter should specify whether central node will be used as a worker node (should install loki plugin on it) -> The loki plugin installation can be done as a separate role and then included in central node and worker nodes.

6. Create ansible role(worker_node) which will edit necessary configs and start services from docker-compose required to expose and ship logs to the centra_node. Here we want also 2 same setups as in central_node:
   a) reverse_proxy: true -> then we setup caddy reverse proxy and expose via subdomains
   b) no reverse_proxy -> then we go for the plain IP template
   Create docker compose with 3/2 services: (caddy for proxy setup),nodeexporter,cadvisor
   After the containers are sucessfuly launched delegate task to central_node (ansible host passed as argument to role), which will add the necessary 2 scrape endpoints to prometheus config (check for already established modules, if not possible, add to YAML and reload prometheus config)
   In verify do same verification as in central_node role.

7. Prepare the finall playbook and hosts file with two groups central_node and worker_nodes and specific variables for them(default values specified on roles, specific values in group_vars)

# Test

1. Test everything on free AWS infra.

# Ideas

1. Port the stack to kubernetes. Drop ansible and docker-compose and focus on helm charts which will allow replication of same services. What about cluster provisioning? (EKS, also terraform?). What about updates/gitops -> argocd?

## Docker container running systemd on host systemd version 248+ (for testing ansible roles which modify systemd services)

### In order to make the --cgroupns=host default, specify following options in `/etc/docker/daemon.json`

```json
{
  "default-cgroupns-mode": "host"
}
```

In such case you can drop the --cgroupns=host flag.

```bash
docker run --detach --privileged \
--volume=/sys/fs/cgroup:/sys/fs/cgroup \
--volume=`pwd`:/testing \
--cgroupns=host \
geerlingguy/docker-rockylinux8-ansible
```

```bash
docker exec --tty [container_id] \
env TERM=xterm ansible-playbook \
/testing/playbook.yml
```

## Run ansible against local docker containers(containers as ansible hosts)

For example for container started with `--name=mycontainer` (or just few letters of container id)

_Note the comma at the end!_

```bash
ansible all -i 'mycontainer,' -c docker -m ping
```
