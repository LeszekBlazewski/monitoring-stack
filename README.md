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

[X] 4. Write full docker-compose file based on https://github.com/stefanprodan/dockprom (check documentation for upgrades, and how this should be started) + add Loki to the compose file. Everything should be running inside one docker network. (What about nodeexporter?)

5. Create ansible role (central_node) which will edit necessary configs and start services from docker-compose required to run the monitoring infrastructure. Required services should be exposed to outside via caddy on TLS on different subdomains. Also a parameter should specify whether central node will be used as a worker node (should install loki plugin on it) -> The loki plugin installation can be done as a separate role and then included in central node and worker nodes.

Current setup for proxy will work only if the ansbile target host has domain setup. What to do if someone does not has a domain and operates only by ip? We switch to ports in docker compose

6. Create ansible role(worker_node) which will edit necessary configs and start services from docker-compose required to expose and ship logs to the worker_node (docker loki plugin, cAdvisor, nodeExporter, caddy). -> only port 80 and 443 should be exposed with caddy, and needed services should be available via paths like /cadvisor /nodeexporter etc. This role if has loki enable should also use variable loki_url to set it inside docker daemon.json. (maybe a check for loki url might be needed, to make sure the central_node is available and listening for logs)

7. Prepare the finall playbook and hosts file with two groups central_node and worker_nodes and specific variables for them(default values specified on roles, specific values in group_vars)

# Test

# Ideas

0. Store credentials in ansible vault

1. Test everything on free AWS infra.

2. After finish, create terraform templates to provision basic ubuntu20.4, debian10 and rocky8 machines where ubuntu20.4 will be the central_node and other workers, which will be provisioned by ansible. + create template for route 53 with adequate DNS records for the machines. Will terragrunt be needed?

3. Port the stack to kubernetes. Drop ansible and docker-compose and focus on helm charts which will allow replication of same services. What about cluster provisioning? (EKS, also terraform?). What about updates/gitops -> argocd?

4. Add sample rules for logs in loki (send to alertmanager)

## Docker container running systemd on host systemd version 248+ (for testing ansible roles which modify systemd services)

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
