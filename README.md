# Ansible role for Docker containers and image build
This role is intended to be included into playbooks for deploying containerized services.  The role makes use of group_vars to build a container image from a specified git repo and then starts the container using the newly created image.

## Example using group_vars
This is an example of how to use this role to deploy the Ubiquiti Unifi Admin controller and a Prometheus metrics collector for Unifi Admin

### docker_containers
Uses the ansible docker_container module to deploy containers with specified arguments.  SystemD services are also created for each container.
```
docker_containers: 
  unifi:
    description: "Unifi Admin Controller"
    image: linuxserver/unifi:unstable
    network_mode: host
    ports: []
    volumes:
      - '{{ data_mount_root }}/{{ configs_directory }}/unifi:/config'
    env:
      PUID: '0'
      PGID: '0'
  unifi_exporter:
    description: "Prometheus Metrics Collector Unifi"
    image: unifi_exporter
    command: "-config.file /etc/unifi_exporter/config.yml"
    network_mode: host
    volumes:
      - '{{ media_root }}/configs/unifi_exporter:/etc/unifi_exporter'
```
### docker_build_images
Uses the Ansible docker_image module to build a docker image using the specified repo and additional arguments.  Currently builds the image on the host which will run the container, but can be easily modified to push to a desired docker hub.
```
docker_build_images:
  unifi_exporter:
    repo: "https://github.com/mdlayher/unifi_exporter.git"
```
## Included as part of a playbook and service role
The following **ansible-unifi** role uses this **ansible-docker** role to build the container image for the unifi-exporter service (prometheus exporter for unifi AP data) and run the Unifi-Admin and the Unifi-Exporter containers.  The role itself creates config files on the docker host that are used by the Unifi-Exporter service and runs additional setup steps required by the Unifi-Admin service.

Together, the roles deploy the complete containerized services and related configuration.

Below are the key components of the role.  You may check out the full role here:  [https://github.com/ajanis/ansible-unifi](https://github.com/ajanis/ansible-unifi)

**NOTE:** You will need Prometheus set up somewhere to use the Unifi-Exporter.  You may want to look at [This InfluxDB Role](https://github.com/ajanis/ansible-influxdb) for deploying InfluxDB + Prometheus, which also includes the required Prometheus configuration for the Unifi-Exporter.


### Sample Playbook
```
- name: Deploy Unifi Controller
  hosts: unifi
  become: True
  tasks:
    - import_role:
        name: docker
    - import_role:
        name: unifi
```
### Main task for Unifi-Admin
```
- name: Ensure Unifi Data Directory Exists
  file:
    state: directory
    path: /data/configs/unifi/data
    owner: root
    group: root

- name: Install SSL Key
  copy:
    content: "{{ vault_prettybaked_com_ssl_private_key }}"
    dest: "{{ ssl_private_key }}"
    owner: root
    group: root
    mode: 0640

- name: Install SSL Certificate Chain
  copy:
    content: "{{ vault_prettybaked_com_ssl_certificate }}"
    dest: "{{ ssl_certificate }}"
    owner: root
    group: root
    mode: 0640

- name: Create SSL Import Script
  template:
    src: unifi_ssl_import.sh.j2
    dest: /data/configs/unifi/unifi_ssl_import.sh
    mode: 0775
    owner: root
    group: root

- name: Build SSL Keystore for Unifi Admin
  shell: /data/configs/unifi/unifi_ssl_import.sh >> /var/log/docker_unifi_ssl_upgrade.log
  args:
    executable: /bin/bash
  notify: restart_docker_unifi
```
### Unfi-Exporter tasks
```
- name: Generate Unifi Prometheus Collector config file
  template:
    src: unifi_exporter_config.yml.j2
    dest: '{{ media_root }}/configs/unifi_exporter/config.yml'
  notify: restart unifi_exporter
```
### Unifi-Exporter Config Template
```
listen:
  address: :9130
  metricspath: /metrics
unifi:
  address: {{ unifi_admin_url }}
  username: {{ unifi_admin_user }}
  password: "{{ unifi_admin_password }}"
  site: {{ unifi_admin_site | default('default') }}
  insecure: true
  timeout: 5s
```
### systemd container service handlers
```
- name: restart_docker_unifi
  service:
    name: docker-unifi
    state: restarted
- name: restart unifi_exporter
  systemd:
    name: docker-unifi_exporter
    state: restarted
    enabled: yes
    daemon_reload: yes
```

