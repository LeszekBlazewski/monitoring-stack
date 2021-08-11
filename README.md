# monitoring-stack

Sample monitoring stack (logs + metrics) build based on:

- Prometheus - metrics db
- Grafana - logs and metrics display
- cAdvisor - container metrics
- NodeExporter - docker host metrics
- AlertManager - alerts based on metrics db
- Loki - container logs aggregation
- caddy - reverse proxy + auto TLS

Based on: https://github.com/stefanprodan/dockprom extended with Loki, wrapped with ansible for full server provisioning.

## Description

There are times when you find yourself running your container workload on different virtual machines not in cloud providers and there aren't any state of the art outside of the box monitoring solutions available. This is a simple mashup of ansible roles for provisioning and deploying full working central monitoring solution running via docker-compose which is able to scrape external resources. Ansible was mainly used in order to ease the process of installing necessary dependencies and configuring the services to run properly.

_This isn't rocket science, all credits go to authors of all the lovely tools used in this repository._

## How was this build?

We assume that the workload will be running via docker containers.

This setup supports one central monitoring node which is responsible for scraping all the targets via prometheus and aggregating logs from different sources utilizing loki. Of course the central node also supports self scrapping so it can be used to run your workload. The other part consists of external nodes called exporters which export monitoring metrics of their workload via nodeexporter and cadvisor. Therefore you are able to monitor metrics and logs of multiple machines running your workload in one place.

## How to check it out quickly?

**If you want to see logs provided by loki data source in grafana**

[Install and configure loki-docker plugin.](https://grafana.com/docs/loki/latest/clients/docker-driver/)

Loki url should be set to: `http://localhost:3100/loki/api/v1/push`

Or better run loki-docker-plugin ansible role from this repo on localhost which will take care of everything. See below for details.

```bash
cd docker-localhost && docker-comopose up -d
```

Corresponding services are available at following urls:

| Service      | url                           |
| ------------ | ----------------------------- |
| prometheus   | http://prometheus.localhost   |
| alertmanager | http://alertmanager.localhost |
| grafana      | http://grafana.localhost      |

## Full Ansible setup description

All the roles(both central and exporter nodes) where build in such a way that they setup two main scenarios:

1.  **Setup with reverse-proxy running on the nodes** -> this should be used if you would like to access your monitoring infrastructure via domain name on subdomains such as grafana.yourdomain.com etc.

2.  **Setup without reverse-proxy** -> this should be used if you would like to access your monitoring infrastructure via ip and ports, for example 1.2.3.4:9090 etc.

### Playbooks && hosts

Based on the description we can easily grasp that we will be working with one central_node host and multiple exporter_nodes, therefore the [hosts](./hosts) file was structured this way.

The [provision.yml](./provision.yml) playbook is responsible for hardening the server security and installing all the necessary dependencies to run the services.

The [deploy.yml](./deploy.yml) playbook is responsible for setting up the configuration of services and starting them on target hosts.

### Roles variables for correct setup

**For variables used by external roles and further customization please check their sources.** All the variables listed down below are presented in such a way that they get you a fully working setup in both scenarios.

#### role: geerlingguy.pip variables

Pip packages are required in order to allow ansible to manage docker containers on target host. **This role should be installed on all of the hosts both central and exporter nodes**.

```yml
pip_install_packages:
  - name: docker
  - name: six
  - name: docker-compose
```

#### role: geerlingguy.docker variables

ansible_user in docker_users will allow executing docker commands without the need to become root user.**This role should be installed on all of the hosts both central and exporter nodes**

```yml
docker_users: ["{{ ansible_user }}"]
```

#### role: loki-docker-plugin

Installs the [loki docker plugin](https://grafana.com/docs/loki/latest/clients/docker-driver/) and configures the docker daemon to use it by default. **It's important that this role runs before starting any containers, since already created containers won't pick up the new log driver.**

| variable                   | value                        | description                                                                                                                                                                           |
| -------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| loki_url                   | http(s)://(domain/ip):(port) | Required variable, should be provided as base url where loki is accessible                                                                                                            |
| skip_loki_url_verification | true/false (default: false)  | When setting up the servers for first time your loki instance will not be available so make sure to set it up to true. Once the central node is provisioned it should be set to true. |
| daemon_options             | dictionary                   | /etc/docker/daemon.json config, check [default role variables](./roles/loki-docker-plugin/defaults/main.yml) for structure.                                                           |

#### role: geerlingguy.firewall variables

By default this role will allow **only** ingress traffic on ports 22, 25, 80 and 443.

##### Ip based setup

If you are running the playbook with ip based setup, remember to state ports on which your services will be exposed. This must go in sync with [role: central-node](#role:-central-node-variables) and [role: exporter-node](#role:-exporter-node-variables) respectively on the hosts you are running your play on. For example:

```yml
firewall_allowed_tcp_ports:
  - "22"
  - "9090"
  - "9093"
  - "9091"
  - "3000"
  - "3100"
```

##### Subdomain setup

Only port 80 and 443 are necessary since, the reverse proxy will handle all the redirects.

```yml
firewall_allowed_tcp_ports:
  - "22"
  - "80"
  - "443"
```

**Remember to add 22 to not lock yourself out!**

#### role: central-node variables

This role is responsible for configuring the central monitoring services and running them on target host via docker compose.

| variable             | value                       | description                                                                                                                                         |
| -------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| monitoring_directory | valid unix path             | where to place the config files and docker-compose, the dir will be created if necessary.                                                           |
| reverse_proxy_setup  | true/false (default: true)  | Should reverse proxy be setup in front of the monitoring services. Set to **true** for subdomains setup and **false** for ip and ports setup.       |
| services_endopoints  | dictionary of urls or ports | Dictionary of urls or ports where services will be available, check [default role variables](./roles/central-node/defaults/main.yml) for structure. |
| slack_config         | dictionary                  | check [default role variables](./roles/central-node/defaults/main.yml) for structure and prometheus-alertmanager integration for correct setup.     |

**Note on services_endopoints**

Remember that when running ip based setup, the ports must be in sync with the firewall rules. When using `reverse_proxy_setup: true` the endpoints should be setup to full subdomain addresses, for example:

```yml
services_endopoints:
prometheus: "prometheus.mydomain.com"
grafana: "grafana.mydomain.com"
```

#### role: exporter-node variables

This role is responsible for configuring the exporter services and running them on target host via docker compose. The metrics will then be scrapped by prometheus.

| variable            | value                       | description                                                                                                                                           |
| ------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| exporters_directory | valid unix path             | where to place the config files and docker-compose, the dir will be created if necessary.                                                             |
| reverse_proxy_setup | true/false (default: true)  | Should reverse proxy be setup in front of the monitoring services. Set to **true** for subdomains setup and **false** for ip and ports setup.         |
| exporters_endpoints | dictionary of urls or ports | Dictionary of urls or ports where exporters will be available, check [default role variables](./roles/exporter-node/defaults/main.yml) for structure. |

**Note on exporters_endpoints**

Same rules apply as in the central-node role.

#### Scraping new exporter nodes

Check the tasks in [deploy.yml play](./deploy.yml) where we register new targets to scrape in central-node prometheus service. They take advantage of central_node host variables and delegate the task to change the scrape config and then reload prometheus. This will only work when prometheus is started with --web.enable-lifecycle flag which is passed in in the corresponding docker-compose service.

## Molecule

Creating this repository was also a good opportunity to take advantage of TDD when developing central-node, exporter-node and loki-docker-plugin roles, which are covered by molecule tests and allow testing on different target systems. The roles also use specific docker containers which are able to run systemd services, so we can test different setups. After navigating to any of the directories in `roles/` folder, simply issue the `molecule $COMMAND` to converge, verify etc.
