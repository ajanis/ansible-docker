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

## Deploying Telegraf
You may also wish to include this [Telegraf](https://github.com/ajanis/ansible-telegraf) role, which will configure the Telegraf service along with the SNMP MIBS + Telegraf config needed for polling Unifi access point metrics.

**NOTE:** You will need InfluxDB set up somewhere to use the Telegraf + SNMP exporter. You may want to look at [This InfluxDB Role](https://github.com/ajanis/ansible-influxdb) for deploying InfluxDB + Prometheus. 

### Updated playbook
```
---
- name: Deploy Unifi Controller
  hosts: unifi
  become: True
  tasks:
    - import_role:
        name: docker
    - import_role:
        name: unifi
    - import_role:
        name: telegraf
```

### Unifi role group_vars for Telegraf
```
telegraf_plugins_extra:
  - name: docker
    options:
      endpoint: "unix:///var/run/docker.sock"
      timeout: "5s"
      perdevice: "true"
      total: "true"
  - name: snmp
    options:
      name: "snmp.UAP"
      agents:
        - "192.168.0.101"
        - "192.168.0.102"
        - "192.168.0.103"
      interval: "10s"
      timeout: "10s"
      retries: 3
      version: 1
      community: "public"
      max_repetitions: 1
  - name: snmp.field
    options:
      is_tag: "true"
      name: "sysName"
      oid: "RFC1213-MIB::sysName.0"
  - name: snmp.field
    options:
      name: "sysObjectID"
      oid: "RFC1213-MIB::sysObjectID.0"
  - name: snmp.field
    options:
      name: "sysDescr"
      oid: "RFC1213-MIB::sysDescr.0"
  - name: snmp.field
    options:
      name: "sysContact"
      oid: "RFC1213-MIB::sysContact.0"
  - name: snmp.field
    options:
      name: "sysLocation"
      oid: "RFC1213-MIB::sysLocation.0"
  - name: snmp.field
    options:
      name: "sysUpTime"
      oid: "RFC1213-MIB::sysUpTime.0"
  - name: snmp.field
    options:
      name: "unifiApSystemModel"
      oid: "UBNT-UniFi-MIB::unifiApSystemModel"
  - name: snmp.field
    options:
      name: "unifiApSystemVersion"
      oid: "UBNT-UniFi-MIB::unifiApSystemVersion"
  - name: snmp.field
    options:
      name: "memTotal"
      oid: "FROGFOOT-RESOURCES-MIB::memTotal.0"
  - name: snmp.field
    options:
      name: "memFree"
      oid: "FROGFOOT-RESOURCES-MIB::memFree.0"
  - name: snmp.field
    options:
      name: "memBuffer"
      oid: "FROGFOOT-RESOURCES-MIB::memBuffer.0"
  - name: snmp.field
    options:
      name: "memCache"
      oid: "FROGFOOT-RESOURCES-MIB::memCache.0"
  - name: snmp.table
    options:
      oid: "IF-MIB::ifTable"
  - name: snmp.table.field
    options:
      is_tag: "true"
      oid: "IF-MIB::ifDescr"
  - name: snmp.table
    options:
      oid: "UBNT-UniFi-MIB::unifiRadioTable"
  - name: snmp.table.field
    options:
      is_tag: "true"
      oid: "UBNT-UniFi-MIB::unifiRadioName"
  - name: snmp.table.field
    options:
      is_tag: "true"
      oid: "UBNT-UniFi-MIB::unifiRadioRadio"
  - name: snmp.table
    options:
      oid: "UBNT-UniFi-MIB::unifiVapTable"
  - name: snmp.table.field
    options:
      is_tag: "true"
      oid: "UBNT-UniFi-MIB::unifiVapName"
  - name: snmp.table.field
    options:
      is_tag: "true"
      oid: "UBNT-UniFi-MIB::unifiVapRadio"
  - name: snmp.table
    options:
      oid: "UBNT-UniFi-MIB::unifiIfTable"
  - name: snmp.table.field
    options:
      is_tag: "true"
      oid: "UBNT-UniFi-MIB::unifiIfName"
  - name: snmp.table
    options:
      oid: "FROGFOOT-RESOURCES-MIB::loadTable"
  - name: snmp.table.field
    options:
      is_tag: "true"
      oid: "FROGFOOT-RESOURCES-MIB::loadDescr"
  - name: snmp.field
    options:
      name: "snmpInPkts"
      oid: "SNMPv2-MIB::snmpInPkts.0"
  - name: snmp.field
    options:
      name: "snmpInGetRequests"
      oid: "SNMPv2-MIB::snmpInGetRequests.0"
  - name: snmp.field
    options:
      name: "snmpInGetNexts"
      oid: "SNMPv2-MIB::snmpInGetNexts.0"
  - name: snmp.field
    options:
      name: "snmpInTotalReqVars"
      oid: "SNMPv2-MIB::snmpInTotalReqVars.0"
  - name: snmp.field
    options:
      name: "snmpInGetResponses"
      oid: "SNMPv2-MIB::snmpInGetResponses.0"
  - name: snmp.field
    options:
      name: "snmpOutPkts"
      oid: "SNMPv2-MIB::snmpOutPkts.0"
  - name: snmp.field
    options:
      name: "snmpOutGetRequests"
      oid: "SNMPv2-MIB::snmpOutGetRequests.0"
  - name: snmp.field
    options:
      name: "snmpOutGetNexts"
      oid: "SNMPv2-MIB::snmpOutGetNexts.0"
  - name: snmp.field
    options:
      name: "snmpOutGetResponses"
      oid: "SNMPv2-MIB::snmpOutGetResponses.0"
```




