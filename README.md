# Mastery of Container Management

```display
Copyright (C)  2023  JOSÉ DANIEL RODRÍGUEZ CHINCHILLA.
Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.3
or any later version published by the Free Software Foundation;
with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.
A copy of the license is included in the section entitled "GNU
Free Documentation License".
```

## Prerequisites

Before we get started installing the `stacks`, we need to make sure that the following prerequisites are met:
- The `host` machine is running an OS-compatible Linux distribution
- [Docker](https://docker.com/) is installed on the `host` machine. It can be easily installed with this `bash` script:
```bash
curl -sSL https://get.docker.com/ | sh
```

- Docker Compose is installed on the `host` machine

Download the latest `docker-compose` binary:
```bash
curl -s https://api.github.com/repos/docker/compose/releases/latest \
       | grep browser_download_url \
       | grep docker-compose-linux-x86_64 \
       | cut -d '"' -f 4 \
       | wget -qi -
```

Make the binary file executable:
```bash
chmod +x docker-compose-linux-x86_64
```

Move the file to your `PATH` and validate the `docker-compose` version:
```bash
sudo mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose && \
docker-compose --version
```

- Add your user to `docker` group and reboot the `host` machine:
```bash
sudo usermod -aG docker $USER && \
sudo reboot
```

- For subdomains, a FQDN. We are using `beautykatherin.com` for this example
- Clone this repository on your `host` machine
```bash
git clone https://github.com/armory86/Mastery-of-Container-Management.git
```

## Portainer with Traefik
This `stack` contains everything needed to prepare a scalable Linux server ready to manage many web applications in a secured environment. It uses Portainer to manage services and Traefik to route traffic to the right container. It's a very simple and easy-to-use solution for small projects and personal use. It's also a good way to learn how to use [Docker](https://docs.docker.com/desktop/) and `docker-compose`

It is based on the following projects:
- [Portainer](https://portainer.io/)
- [Traefik](https://github.com/traefik/traefik/)

### Features

With Traefik and Portainer BE, you will be able to:

- Reverse proxy exposing only `https:443`
- Simple configuration, clean, lightweight, and forgettable
- Able to manage many domain names and/or subdomains
- Automagically creating validated certificates (with [Let's Encrypt](https://letsencrypt.org/))
- [Docker](https://docs.portainer.io/user/docker) administration from a friendly web interface
- You can get a copy of the [license](https://portainer.io/take-5) for up to 5 nodes of the enterprise edition for free

### Installation and Configuration

To install the `stack`, follow the steps below:

- Create `traefik` network
```bash
docker network create traefik
```

- Enter the cloned directory `stack`
```bash
cd Mastery-of-Container-Management/portaefik
```

- Create `letsencrypt` and `data` directories
```bash
mkdir letsencrypt && \
mkdir -p portainer/data
```

### Traefik Configuration

So! The configuration tells Traefik to:

- Accept `https:443`
- Watch [Docker engine socket](https://docs.docker.com/desktop/extensions-sdk/guides/use-docker-socket-from-backend/) on /var/run/docker.sock for containers to route to frontend
- Use network called `traefik` for containers (so it will not route containers you don't want to route)
- Create a certificate on `host` rules using Let's Encrypt

On your nameserver provider create a **CNAME** entry for `portainer.domain.tld` and `traefik.domain.tld`. So, when you try to access to `https://portainer.domain.tld` everything should be okay
- Here is an example how `DNS` records on Namecheap must be:

![alt text](https://github.com/armory86/Mastery-of-Container-Management/blob/main/screenshots/DNS_records.png?raw=true)

- Edit [docker-compose.yml](portaefik/docker-compose.yml) and change lines `youremail@domain.tld` and `Host=(subdomain.domain.tld)` for both services e.g. `traefik` and `portainer`

- Edit the auth file [users.u](/portaefik/users/users.u) to secure your Traefik dashboard. Put user and password in it. You can generate a password/hash with htpasswd:

```bash
sudo apt install -y apache2-utils && \
htpasswd -nb username password
```

Alternatively, you can generate a password/hash on [htaccesstools.com](http://www.htaccesstools.com/htpasswd-generator/). Paste the resulting output `user:hash` into the file

- Start the `stack` with the `docker-compose` file:
```bash
docker-compose up -d
```

### Troubleshooting

- If you are not sure about what you are doing, you can make some tests with Let's Encrypt staging environment. Just uncomment the lines `- "--certificatesresolvers.letsencryptresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"` and `- "--log.level=DEBUG"` in [docker-compose.yml](portaefik/docker-compose.yml) and start the `stack` using `docker-compose up` (without -d)

- After checking that the certificates are properly generated, don't forget to delete `acme.json` in the `letsencrypt` folder before restarting in the production environment

- Take no time to create an `admin` password for Portainer. Use a very strong password. You can also create a new `admin user` with a different name and delete the one called `admin` for increased security

- You can also access `https://traefik.domain.tld`. Again, please use a strong password and avoid users like `admin`, `root` or any other common usernames

### How to use it ?

So, the only open `ports` Traefik is handling are `http:80` and `https:443`. It's recommended to deny access to any other `ports`. When you want to create a new web app, you have to first write a `docker-compose` file and send it to Portainer (in `Stacks` menu). Alternatively, you can use the `App Templates` menu or start new containers manually.

The important things to remember are:

- For Traefik to route your server, you must connect it to the network called `traefik`
- Even if on the `traefik` network, you can enable or disable the route by adding the label `traefik.enable=false` (or true) to the container
- Do not expose the `port` on your `host`. If your `docker-compose` use the `port` section, remove it and route it to Traefik instead
- Use the label `traefik.http.routers.[servicename].rule=Host` to route a subdomain to your container
- Use the label `traefik.http.services.[servicename].loadbalancer.server.port` to route the container `port` to the frontend rule

### Example

Here is a `docker-compose` template to deploy a simple WordPress site as `stack`:

- Create a **CNAME** entry for `wordpress.domain.tld` to route your subdomain to your server IP on your nameserver provider e.g. Namecheap and edit `Host=(wordpress.domain.tld)`

- Paste this content into Portainer's `Stacks` page:

```bash
version: '3'
services:
  mariadb:
    image: mariadb:latest
    container_name: mariadb
    volumes:
      - 'mariadb_data:/var/lib/mysql'
    environment:
      MARIADB_DATABASE: db_name
      MARIADB_USER: db_user
      MARIADB_PASSWORD: db_user_pass
      MARIADB_RANDOM_ROOT_PASSWORD: 'root_pass'
    networks:
      - traefik
    restart: unless-stopped
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    volumes:
      - 'wordpress_data:/var/www/html'
    depends_on:
      - mariadb
    environment:
      WORDPRESS_DB_HOST: mariadb
      WORDPRESS_DB_USER: db_user
      WORDPRESS_DB_PASSWORD: db_user_pass
      WORDPRESS_DB_NAME: db_name
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.wordpress.loadbalancer.server.port=80"
      - "traefik.http.routers.wordpress.rule=Host(`wordpress.domain.tld`)"
      - "traefik.http.routers.wordpress.entrypoints=websecure"
      - "traefik.http.routers.wordpress.tls.certresolver=letsencryptresolver"
    restart: unless-stopped
networks:
  traefik:
    external: true
volumes:
  mariadb_data:
    driver: local
  wordpress_data:
    driver: local
```

- When you are ready, click on Deploy the `stack` from Portainer. This is where the magic starts. Go to `https://wordpress.domain.tld` it should work using `https` with a valid certificate

Note that the container is not exposing its `port` to the `host`. Instead, you use labels to handle it. The frontend rule tells Traefik on which URL to route that container's `port`, here explains how it works:

![alt text](https://doc.traefik.io/traefik/assets/img/architecture-overview.png)

### Screenshots

#### Portainer
![alt text](https://github.com/armory86/Mastery-of-Container-Management/blob/main/screenshots/screenshot_1.png?raw=true)

#### WordPress
![alt text](https://github.com/armory86/Mastery-of-Container-Management/blob/main/screenshots/screenshot_2.png?raw=true)

#### Traefik
![alt text](https://github.com/armory86/Mastery-of-Container-Management/blob/main/screenshots/screenshot_3.png?raw=true)

## Grafana

A GNU/Linux monitoring solution using Grafana, Prometheus, cAdvisor, and Node-Exporter `stack`. This project aims to provide a comprehensive and user-friendly way to monitor the performance of your OS-compatible Linux distribution. With Grafana's intuitive dashboards, you can easily visualize system metrics collected by Prometheus and cAdvisor, while Node-Exporter provides valuable information about the `host` hardware and operating system.

It is based on the following projects:
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.org/)
- [cAdvisor](https://github.com/google/cadvisor/)
- [NodeExporter](https://github.com/prometheus/node_exporter/)

### Installation and Configuration

To install the `stack`, follow the steps below:

- Enter the cloned directory `stack`
```bash
cd Mastery-of-Container-Management/grafana
```

- Create `data` directory and change the ownership of the `prometheus` and `grafana` folders for a nice and clean installation
```bash
mkdir -p prometheus/data grafana/data && \
sudo chown -R 472:472 grafana/ && \
sudo chown -R 65534:65534 prometheus/
```

- Create a **CNAME** entry for `grafana.domain.tld`

- Edit [docker-compose.yml](grafana/docker-compose.yml) and change lines `Host=(grafana.domain.tld)`

- Start the `stack` with the `docker-compose` file:
```bash
docker-compose up -d
```

This will start all the containers and make them available on the host machine. The following ports are used:

- `grafana`: 3000
- `prometheus`: 9090
- `cadvisor`: 8080
- `node-exporter`: 9100

If you would like to change which targets should be monitored, you can edit the [prometheus.yml](grafana/prometheus/prometheus.yml) file. The targets section contains a list of all the targets that should be monitored by Prometheus. The names defined in the `job_name` section are used to identify the targets in Grafana. The `static_configs` section contains the `IP` addresses of the targets that should be monitored. If you think that the `scrape_interval` value is too aggressive, you can change it to a more suitable value. Actually, they are sourced from the service names defined in the [docker-compose.yml](grafana/docker-compose.yml) file.

- The Grafana dashboard can be accessed by navigating to `grafana.domain.tld` in your browser
- The default username and password are both `admin`; It will prompt you to change the password on the first login
- Credentials can be changed by editing the [.env](grafana/grafana/.env) file

### Add Data Sources and Dashboards

Since Grafana v5 has introduced the concept of provisioning, it is possible to automatically add data sources and dashboards to Grafana. This is done by placing the `datasources` and `dashboards` directories in the [provisioning](grafana/grafana/provisioning) folder. The files in these directories are automatically loaded by Grafana on startup.

If you like to add a new dashboard, simply place the `JSON` file in the [dashboards](grafana/grafana/provisioning/dashboards) directory, and it will be automatically loaded next time Grafana is started

### Install Dashboard from Grafana (Optional)

If you would like to install this dashboard from Grafana, simply follow the steps below:
- Navigate to the dashboards page on [Grafana.com](https://grafana.com/grafana/dashboards/)
- Click on the `Copy ID to Clipboard` button
- Navigate to the `Import` in Grafana's dashboard section
- Paste the ID into the `Import via grafana.com` field
- Click on the `Load` button
- Select `Prometheus` as Data source
- Click on the `Import` button

Or you can follow the steps described in the [Grafana Documentation](https://grafana.com/docs/grafana/latest/dashboards/manage-dashboards/#import-a-dashboard)

- One of my recommendations for a dashboard is [cAdvisor Exporter](https://grafana.com/grafana/dashboards/14282-cadvisor-exporter/)

### Screenshots

![alt text](https://github.com/armory86/Mastery-of-Container-Management/blob/main/screenshots/screenshot_4.png?raw=true)
![alt text](https://github.com/armory86/Mastery-of-Container-Management/blob/main/screenshots/screenshot_5.png?raw=true)

## License

This project is licensed under the [GNU Free Documentation License v1.3](https://www.gnu.org/licenses/fdl-1.3.html) - see the [LICENSE](LICENSE) file for more details.
