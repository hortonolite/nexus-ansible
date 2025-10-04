# Nexus Repository Manager + Nginx + Let’s Encrypt SSL in Docker Compose Ansible playbook  

This Ansible playbook installs the Docker Compose plugin with Sonatype Nexus Repository Manager OSS 3.x, Nginx reverse proxy and Let's Encrypt SSL with auto-renewal.

It was tested on Ubuntu Focal and Debian 12, but should work on other Linux distributions with systemd and Docker Compose support.


## Deployment  
Generate Ansible configs by running the following command
```shell
ansible-playbook ansible/0_generate_configs.yml
```

Edit `ansible/inventory.ini` and specify your Nexus server address and username, for example:
```ini
[nexus]
nexus ansible_host=ubuntu@nexus.example.org
```

Edit `ansible/group_vars/all.yml` and specify your Nexus server DNS name and other parameters, for example:
```yaml
# === Nexus
nexus_docker_image: sonatype/nexus3:latest
nexus_docker_compose_location: /srv/nexus
nexus_data_directory: "{{ nexus_docker_compose_location }}/nexus-data"
nexus_autoconfiguration: false


# === Portainer
portainer_enabled: false
portainer_image: portainer/portainer-ce:latest
portainer_host_port: 9443
portainer_data_directory: "{{ nexus_docker_compose_location }}/portainer-data"
portainer_use_nginx_proxy: false
portainer_server_name: docker.chnch.us
# Example:
# portainer_server_alias:
#   - d.chnch.us
portainer_server_alias: []
portainer_backend_port: 9000


# === Nginx
nexus_nginx_docker_image: nginx:latest
nexus_backend_name: nexus
nexus_host_port: 8081
nexus_backend_port: 8081
nexus_server_name: nexus.example.org
# Example:
# nexus_server_alias:
#   - repo.example.org
nexus_server_alias: []


# == Let's Encrypt
nexus_certbot_email: admin@example.org
nexus_certbot_renew_cron_hour: 3
nexus_certbot_renew_cron_minute: 20
nexus_certbot_staging: true


# === Docker Compose
# Docker Compose v2 plugin is installed with Docker Engine packages
```

Run Ansible playbook
```shell
cd ./ansible
ansible-playbook -i inventory.ini 1_install_docker.yml 2_install_nexus.yml
```

## Configuration  
Deployment parameters are specified in the `ansible/group_vars/all.yml` configuration file.  

Specify the email address, which will be supplied to Let's Encrypt while requesting SSL certificate, in the `nexus_certbot_email` parameter.  

Specify hour and minute of the daily Let's Encrypt certificate renewal cron job in the `nexus_certbot_renew_cron_hour` and `nexus_certbot_renew_cron_hour` parameters.  

If you wish to test Let's Encrypt SSL certificates issuing, set  `nexus_certbot_staging: true` - it helps to avoid hitting Let's Encrypt certificate issuing rate limits while running certbot multiple times.  

The `nexus_backend_name` and `nexus_backend_port` parameters are used in Nginx configuration file to tell Nginx reverse proxy how to connect to its backend - Nexus.

Use `nexus_host_port` to control which host port the Nexus container publishes without changing the container-side port. For example, setting `nexus_host_port: 8082` keeps the container listening on the `nexus_backend_port` value (default 8081) while exposing the service externally on port 8082. The Nexus data volume path can be tuned with `nexus_data_directory`, which defaults to `<nexus_docker_compose_location>/nexus-data` so you can relocate persistent storage if desired.

Portainer CE can be deployed alongside Nexus to provide a web UI for managing the Docker host. Enable it by setting `portainer_enabled: true` in `ansible/group_vars/all.yml`. You can further customize the image via `portainer_image`, change the direct HTTPS listener with `portainer_host_port`, or adjust the data path with `portainer_data_directory`. When `portainer_use_nginx_proxy: false` (default), the Portainer container exposes its built-in HTTPS endpoint on `https://<host>:<portainer_host_port>` (default `9443`) and persists data under the configured directory.

If you prefer to serve Portainer with a trusted certificate and friendly hostname (for example `docker.chnch.us` or `d.chnch.us`), set `portainer_use_nginx_proxy: true` and supply the desired names through `portainer_server_name` and `portainer_server_alias`. With those settings Nginx terminates TLS on port 443 using the same Let's Encrypt workflow as Nexus, proxies requests to Portainer over the internal HTTP port defined by `portainer_backend_port` (default `9000`), and the docker-compose deployment automatically includes the Portainer hostnames in the certificate request. Provide any additional hostnames as a YAML list in `portainer_server_alias` so each entry is added to the TLS Subject Alternative Name list.

The `nexus_server_name` and `nexus_server_alias` parameters instruct Nginx to forward traffic for the specified hostnames to Nexus. Populate `nexus_server_alias` with a YAML list when you need to advertise multiple additional hostnames, or leave it empty if you only use the primary name.

The Nexus deployment Ansible role is able to execute a script which configures Nexus using its REST API. If you wish to use this feature, you may edit the `ansible/roles/install_nexus/templates/nexus-autoconfiguration.sh.j2` autoconfiguration script template and set `nexus_autoconfiguration: true` in `ansible/group_vars/all.yml` before you run the deployment.

Below is an example of the autoconfiguration script which creates a Docker proxy repository:
```shell
#!/bin/sh

curl -u admin:{{ nexus_password }} -k -H 'Content-Type: application/json' 'https://{{ nexus_server_name }}/service/rest/v1/script' -d '{"name": "CreateDockerProxy","type": "groovy","content": "repository.createDockerProxy('\''docker-proxy'\'', '\''https://registry-1.docker.io'\'', '\''HUB'\'', null, null, null)"}'
curl -X POST -u admin:{{ nexus_password }} -k -H 'Content-Type: text/plain' 'https://{{ nexus_server_name }}/service/rest/v1/script/CreateDockerProxy/run'
```

Please note that autoconfiguration can only be performed on the first run of the Nexus deployment role, because Nexus admin password is only known at the first start of Nexus.


## References  
This work is partially based on the following sources.  

[1] How To Secure a Containerized Node.js Application with Nginx, Let's Encrypt, and Docker Compose // https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose  
[2] Setup Nexus repository manager with Nginx reverse proxy and Let’s Encrypt SSL certificate on Docker // https://medium.com/@numb95/setup-nexus-repository-manager-with-nginx-reverse-proxy-and-lets-encrypt-ssl-certificate-on-docker-1c1b05988ce3


## Ansible Role Execution Example
```shell
$ ansible-playbook -i inventory.ini 1_install_nexus.yml
[WARNING]: Found both group and host with same name: nexus

PLAY [nexus] **********************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Create /srv/nexus] ******************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over the docker-compose file] *************************************************************************************
changed: [nexus]

TASK [install_nexus : Make up server alias parameters string] *********************************************************************************************
skipping: [nexus]

TASK [install_nexus : Update server aliases in certbot command in the docker-compose file] ****************************************************************
skipping: [nexus]

TASK [install_nexus : Enable Lets Encrypt staging mode in certbot command in the docker-compose file] *****************************************************
skipping: [nexus]

TASK [install_nexus : Disable Lets Encrypt staging mode in certbot command in the docker-compose file] ****************************************************
changed: [nexus]

TASK [install_nexus : Create Nexus data directory /srv/nexus/nexus-data] **********************************************************************************
changed: [nexus]

TASK [install_nexus : Create Nginx configuration directory /srv/nexus/nginx-conf] *************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over Nginx configuration file "/srv/nexus/nginx.conf"] ************************************************************
ok: [nexus]

TASK [install_nexus : Create Nginx dhparam directory /srv/nexus/dhparam] **********************************************************************************
ok: [nexus]

TASK [install_nexus : Generate dhparam key] ***************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Create Lets Encrypt directories "{{ nexus_docker_compose_location }}/{{ item }}"] ***************************************************
ok: [nexus] => (item=letsencrypt)
ok: [nexus] => (item=web-root)

TASK [install_nexus : Create dummy certificate for "nexus.example.org"] ************************************************************************************
ok: [nexus] => (item=mkdir -p "/srv/nexus/letsencrypt/live/nexus.example.org")
ok: [nexus] => (item=docker compose run --rm --entrypoint "openssl req -x509 -nodes -newkey rsa:2048 -days 1 -keyout '/etc/letsencrypt/live/nexus.example.org/privkey.pem' -out '/etc/letsencrypt/live/nexus.example.org/fullchain.pem' -subj '/CN=localhost'" certbot)

TASK [install_nexus : Check if Nexus properties file exists] **********************************************************************************************
ok: [nexus]

TASK [install_nexus : Create "/srv/nexus/nexus-data/etc" directory] ***************************************************************************************
changed: [nexus]

TASK [install_nexus : Create "/srv/nexus/nexus-data/etc/nexus.properties" file] ***************************************************************************
changed: [nexus]

TASK [install_nexus : Enable Nexus REST API creation features] ********************************************************************************************
changed: [nexus]

TASK [install_nexus : Create /etc/systemd/system] *********************************************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over the systemd service file] ************************************************************************************
ok: [nexus]

TASK [install_nexus : Reload systemd] *********************************************************************************************************************
changed: [nexus]

TASK [install_nexus : Check if Nginx is using dummy certificate] ******************************************************************************************
ok: [nexus]

TASK [install_nexus : Wait for Nginx startup] *************************************************************************************************************
skipping: [nexus]

TASK [install_nexus : Remove dummy certificate for "nexus.example.org"] ************************************************************************************
skipping: [nexus]

TASK [install_nexus : Wait for Certbot to obtain Lets Encrypt certificate for "nexus.example.org"] *********************************************************
skipping: [nexus]

TASK [install_nexus : Restart Nginx container] ************************************************************************************************************
skipping: [nexus]

TASK [install_nexus : Create Lets Encrypt certificate renewal cron job] ***********************************************************************************
ok: [nexus]

TASK [install_nexus : Wait for Nexus to come up] **********************************************************************************************************
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (20 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (19 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (18 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (17 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (16 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (15 retries left).
ok: [nexus]

TASK [install_nexus : Check if Nexus first-time password file exists] *************************************************************************************
ok: [nexus]

TASK [install_nexus : Look up Nexus admin first-time password] ********************************************************************************************
changed: [nexus]

TASK [install_nexus : set_fact] ***************************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over nexus-autoconfiguration.sh] **********************************************************************************
changed: [nexus]

TASK [install_nexus : Execute Nexus autoconfiguration script "nexus-autoconfiguration.sh"] ****************************************************************
changed: [nexus]

TASK [install_nexus : Disable Nexus REST API creation features] *******************************************************************************************
changed: [nexus]

TASK [install_nexus : Restart Nexus container] ************************************************************************************************************
changed: [nexus]

TASK [install_nexus : Please change Nexus admin password after first-time login] **************************************************************************
ok: [nexus] => {
    "msg": "Nexus admin password: 999d4924-6a57-47f9-8ae3-c02144db3674"
}

PLAY RECAP ************************************************************************************************************************************************
nexus                      : ok=29   changed=12   unreachable=0    failed=0    skipped=7    rescued=0    ignored=0  
```