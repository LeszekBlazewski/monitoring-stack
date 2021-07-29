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

1. Selinux provisioning (based on geerlingguy role), ssh config, security hardening etc
2. Firewall configuration (based on geerlingguy role), open ports:
   - 22 (SSH)
   - 80 (HTTP)
   - 443 (HTTPS)
3. Docker installation (based on geerlingguy role)
4. Write full docker-compose file based on https://github.com/stefanprodan/dockprom (check documentation for upgrades, and how this should be started) + add SWAG container or use caddy (with nginx configs) + add Loki to the compose file. Everything should be running inside one docker network.
5. Create ansible role (central_node) which will edit necessary configs and start ONLY services from docker-compose required to run the monitoring infrastructure so (promethes, grafana, alertmanager, loki, swag/caddy). Required services should be exposed to outside via swag/caddy on TLS on different paths for example /prometheus /grafana /loki etc.
6. Create ansible role(worker_node) which will edit necessary configs and start ONLY services from docker-compose required to expose and ship logs to the worker_node (docker loki plugin, cAdvisor, nodeExporter, swag/caddy). -> only port 80 and 443 should be exposed with nginx, and needed services should be available via paths like /cadvisor /nodeexporter etc. This role if has loki enable should also use variable loki_url to set it inside docker daemon.json. (maybe a check for loki url might be needed, to make sure the central_monitoring_node is available)
7. Prepare the finall playbook and hosts file with two groups monitoring_nodes and worker_nodes and specific variables for them(default values specified on roles, specific values in inventory or group_vars)

# Test

1. Test whether geerlingguy role https://github.com/geerlingguy/ansible-role-security works correctly for Debian/Ubuntu and rockylinux. Provision EC2 instance and run this playbook. If it does not work, rewrite the role as my with correct updates and place it in repo.

# Ideas

0. Store credentials in ansible vault

1. Test everything on free AWS infra.

2. After finish, create terraform templates to provision basic ubuntu20.4, debian10 and rocky8 machines where ubuntu20.4 will be the central_node and other workers, which will be provisioned by ansible. + create template for route 53 with adequate DNS records for the machines. Will terragrunt be needed?

3. Port the stack to kubernetes. Drop ansible and docker-compose and focus on helm charts which will allow replication of same services. What about cluster provisioning? (EKS, also terraform?). What about updates/gitops -> argocd?
