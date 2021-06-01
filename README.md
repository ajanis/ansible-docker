# Ansible role for Docker containers and image build
This role is intended to be included into playbooks for deploying containerized services.  The service role should include group_vars defining the container run parameters and container image parameters (including git repository), along with any container-specific prerequisites and config files.  The role is intended to make it easier to deploy several containerized services on a specified host.

## Example
The following example **ansible-mediaservices** role uses this **ansible-docker** role to build the container image 
for a plex media server and suporting services. (Plex server container, Tautulli, and Varken for Plex, Tautulli, 
Sonarr, Radarr and SABNZBD metrics 
collection)  

Below are the key configs of the role.  You may check out the full role here:  [https://github.
com/ajanis/ansible-mediaservices]
(https://github.com/ajanis/ansible-mediaservices)

**NOTE:** You will need Prometheus set up somewhere to use the Unifi-Exporter.  You may want to look at [This InfluxDB Role](https://github.com/ajanis/ansible-influxdb) for deploying InfluxDB + Prometheus, which also includes the required Prometheus configuration for the Unifi-Exporter.

### group_vars used in service role
Calls the docker role with the following group_vars to deploy a plex server container, nvidia configurations 
for hardware decoding, and supporting containers.

#### group_vars/plexservers/vars.yml
```yaml
docker_install_nvidia: True
docker_install_community_edition: True

docker_containers:
  plex:
    description: "Plex Media Server"
    image: plexinc/pms-docker:plexpass
    network_mode: host
    runtime: nvidia
    privileged: yes
    recreate: "{{ docker_recreate|default(false) }}"
    state: started
    pull: yes
    restart-policy: unless-stopped
    ports: []
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/plex:/config'
      - '{{ media_root }}/ssl:/ssl'
      - '{{ media_root }}/tv:/tvshows'
      - '{{ media_root }}/movies:/movies'
      - '{{ media_root }}/images:/photos'
      - '/transcode:/transcode'
    devices:
      - '/dev/dri/card0:/dev/dri/card0'
      - '/dev/dri/renderD128:/dev/dri/renderD128'
    env:
      TZ: '{{ timezone }}'
      NVIDIA_DRIVER_CAPABILITIES: compute,video,utility
      NVIDIA_VISIBLE_DEVICES: all
      PLEX_UID: '{{ media_user_uid }}'
      PLEX_GID: '{{ media_user_gid }}'
      ALLOWED_NETWORKS: '10.0.0.0/8,172.16.0.0/16,192.168.0.0/16'
      CHANGE_CONFIG_DIR_OWNERSHIP: 'false'

  tautulli:
    description: "Tautulli - Plex Media Server Usage Metrics"
    image: tautulli/tautulli:nightly
    network_mode: host
    state: started
    pull: yes
    recreate: "{{ docker_recreate|default(false) }}"
    restart-policy: unless-stopped
#    ports:
#      - 8181:8181
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/tautulli:/config'
      - '{{ media_root }}/configs/plex/Library/Application Support/Plex Media Server/Logs:/plex_logs:ro'
    env:
      TZ: '{{ timezone }}'
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      ADVANCED_GIT_BRANCH: 'nightly'

  varken:
    description: "Varken Metrics Collector"
    image: boerderij/varken:develop
    state: started
    recreate: "{{ docker_recreate|default(false) }}"
    network_mode: host
    restart-policy: unless-stopped
    pull: yes
    volumes:
      - '{{ media_root }}/configs/varken:/config'
    env:
      TZ: '{{ timezone }}'
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
```
#### group_vars/nzbservices/vars.yml
```yaml
docker_containers:
  qbittorrent:
    description: "Torrent Download Service"
    image: linuxserver/qbittorrent
    restart_policy: unless-stopped
    recreate: "{{ docker_recreate|default(false) }}"
    network_mode: bridge
    pull: yes
    ports:
      - 8082:6881
      - 8082:6881/udp
      - 9000:8080
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/qbittorrent:/config'
      - '{{ torrent_complete_download_directory }}:/downloads'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
  sabnzbd:
    description: "NZB Download Service"
    image: linuxserver/sabnzbd:latest
    restart_policy: unless-stopped
    recreate: "{{ docker_recreate|default(false) }}"
    network_mode: bridge
    pull: yes
    ports:
      - 8181:8080
      - 8143:9090
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/sabnzbd:/config'
      - '{{ usenet_incomplete_download_directory }}:/incomplete-downloads'
      - '{{ usenet_complete_download_directory }}:/downloads'
      - '{{ usenet_watch_directory }}:/watch'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
  sonarr:
    description: "NZB Search Engine and Library Manager for TV Shows"
    image: linuxserver/sonarr:develop
    restart_policy: unless-stopped
    network_mode: bridge
    recreate: "{{ docker_recreate|default(false) }}"
    ports:
      - 8180:8989
    pull: yes
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '/dev/rtc:/dev/rtc:ro'
      - '{{ media_root }}/configs/sonarr:/config'
      - '{{ usenet_complete_download_directory }}:/downloads'
      - '{{ media_root }}/tv:/tv'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'
  radarr:
    description: "NZB Search Engine and Library Manager for Movies"
    image: linuxserver/radarr:latest
    restart_policy: unless-stopped
    network_mode: bridge
    pull: yes
    recreate: "{{ docker_recreate|default(false) }}"
    ports:
      - 8183:7878
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/radarr:/config'
      - '{{ usenet_complete_download_directory }}/:/downloads'
      - '{{ media_root }}/movies:/movies'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'

  ombi:
    description: "User Requests for Media Server"
    image: linuxserver/ombi:latest
    restart_policy: unless-stopped
    network_mode: bridge
    pull: yes
    recreate: "{{ docker_recreate|default(false) }}"
    ports:
      - 3579:3579
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '/dev/rtc:/dev/rtc:ro'
      - '{{ media_root }}/configs/ombi:/config'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'
  bazarr:
    description: "Subtitle Companion App for Sonarr/Radarr"
    image: linuxserver/bazarr:latest
    restart_policy: unless-stopped
    network_mode: bridge
    pull: yes
    recreate: "{{ docker_recreate|default(false) }}"
    ports:
      - 6767:6767
    volumes:
      - '{{ media_root }}/configs/bazarr:/config'
      - '{{ media_root }}/tv:/tv'
      - '{{ media_root }}/movies:/movies'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'
```
### Sample group_vars for grafana/influx/timescale config
```yaml
docker_install_community_edition: True
grafana_containerized: True

influx_containerized: True
kapacitor_containerized: True
chronograf_containerized: True

grafana_enable_auth_ldap: True
grafana_enable_auth_anonymous: True
grafana_allow_embedding: True
grafana_auth_anonymous_org_name: 'Main Org.'
grafana_backup_archive_dir: /data/backups/grafana
grafana_enable_smtp: True
grafana_smtp_host: "smtp.gmail.com"
grafana_smtp_port: '587'
grafana_smtp_user: "eddie.heartofgold.galaxy@gmail.com"
grafana_smtp_password: "{{ vault_eddie_gmail_password }}"
grafana_smtp_from_address: "eddie.heartofgold.galaxy@gmail.com"
grafana_smtp_from_name: "Grafana Alerts"
grafana_install_release: 'stable'
grafana_database_host: '10.0.10.149'
grafana_database_type: 'postgres'
grafana_database_user: 'grafana'
grafana_database_name: 'grafana'
grafana_config_uid: 472
grafana_config_gid: 472
grafana_database_user_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          38646361323938643966353964333831626630653836333365626363396161383734633230666566
          6330616431346135313864346266656537383431396366370a653962613334316131623665326361
          63393737653837383665313665316362386531363365336137303134363138396166373665383966
          3230313066303130610a393564396261313336316133643033643263383435313437393165303961
          3064

grafana_database_log_queries: False
grafana_remote_cache: 'redis'
grafana_cache_connstr: 'addr=127.0.0.1:6379,pool_size=100,db=0,ssl=insecure'
grafana_postgres_ssl_mode: 'disable'
#grafana_provider_config: "user=grafana password=pgstats host=10.0.10.149 port=5432 dbname=grafana sslmode=disable"
#grafana_session_provider: postgres
grafana_session_provider: 'redis'
grafana_provider_config: 'addr=127.0.0.1:6379,pool_size=100,db=grafana,ssl=insecure'
grafana_database_max_idle_conn: 4
grafana_database_max_open_conn: 300
grafana_datasource_limit: 5000
grafana_groups_ou: "{{ groups_ou }}"
grafana_openldap_server_bind_dn: "{{ openldap_server_bind_dn }}"
grafana_openldap_server_dc: "{{ openldap_server_dc }}"
grafana_openldap_server_ip: "{{ openldap_server_ip }}"
grafana_openldap_server_rootpw: "{{ openldap_server_rootpw }}"

grafana_ldap_configurations:
  - name: attributes
    options:
      name: "givenName"
      surname: "sn"
      username: "uid"
      member_of: "cn"
      email: "mail"
  - name: group_mappings
    options:
      group_dn: "admin"
      org_role: "Admin"
      grafana_admin: true
      org_id: 1
  - name: group_mappings
    options:
      group_dn: "users"
      org_role: "Viewer"
  - name: group_mappings
    options:
      group_dn: "media"
      org_role: "Viewer"

docker_compose_projects:
  - project_name: metrics
    pull: yes
    definition:

      version: '3.7'
      x-logging: &default-logging
        driver: journald

      networks:
        metrics:
          name: metrics
          driver: bridge
          ipam:
            driver: default
            config:
              - subnet: 172.3.29.0/24

      services:
#       Define influxdb service
        influxdb:
          hostname: influxdb
          image: influxdb:1.8.5
          networks:
            metrics:
              aliases:
                - influxdb
                - influx
          ports:
            - "8086:8086"
          volumes:
            - '/var/lib/influxdb/:/var/lib/influxdb/'
            - '/etc/influxdb/:/etc/influxdb/'
          logging:
            << : *default-logging
            options:
              tag: influxdb
#       Define grafana server service
        grafana-server:
          hostname: grafana
          image: grafana/grafana:latest
          networks:
            metrics:
              aliases:
                - grafana
                - grafana-server
                - stats
          ports:
            - "3000:3000"
          privileged: True
          depends_on:
            - redis
            - influxdb
          logging:
            << : *default-logging
            options:
              tag: grafana
          volumes:
            - "/var/lib/grafana/:/var/lib/grafana"
            - "/etc/grafana/:/etc/grafana/"
            - "/var/log/grafana/:/var/log/grafana"
            - "/etc/localtime:/etc/localtime:ro"
          environment:
            GF_INSTALL_PLUGINS: 'grafana-clock-panel,grafana-simple-json-datasource,briangann-gauge-panel,grafana-polystat-panel,grafana-piechart-panel,ntop-ntopng-datasource,vonage-status-panel,grafana-worldmap-panel,natel-discrete-panel,petrslavotinek-carpetplot-panel'
            PUID: '472'
            GUID: '472'
            DOCKER_CLIENT_TIMEOUT: '300'
          links:
            - influxdb
#       Define redis db service
        redis:
          hostname: redis
          image: redis:latest
          networks:
            metrics:
              aliases:
                - redis
                - redisdb
          ports:
            - 6379:6379
          logging:
            << : *default-logging
            options:
              tag: redis
          volumes:
            - "/var/lib/redis/:/data"
          environment:
            DOCKER_CLIENT_TIMEOUT: '300'
#       Define Unifi-Poller Service
        unifi-poller:
          hostname: unifi-poller
          image: golift/unifi-poller:latest
          volumes:
            - "{{ unifi_poller_config_directory }}:/config"
          networks:
            metrics:
              aliases:
                - unifi-poller
          logging:
            << : *default-logging
            options:
              tag: unifi-poller
          environment:
            DOCKER_CLIENT_TIMEOUT: '300'
          depends_on:
            - redis
            - influxdb
#       Define a Chronograf service
        chronograf:
          hostname: chronograf
          image: chronograf:latest
          environment:
            INFLUXDB_URL: http://influxdb:8086
            KAPACITOR_URL: http://kapacitor:9092
            RESOURCES_PATH: "/usr/share/chronograf/resources"
          volumes:
            - '/etc/localtime:/etc/localtime:ro'
            - '/var/lib/chronograf/data/:/var/lib/chronograf/'
          ports:
            - "8888:8888"
          links:
            - influxdb
            - kapacitor
          logging:
            << : *default-logging
            options:
              tag: chronograf
          networks:
            metrics:
              aliases:
                - chronograf
          command: ["chronograf", "--influxdb-url=http://influxdb:8086"]
        # Define a Kapacitor service
        kapacitor:
          hostname: kapacitor
          image: kapacitor:latest
          environment:
            KAPACITOR_HOSTNAME: kapacitor
            KAPACITOR_INFLUXDB_0_URLS_0: http://influxdb:8086
          links:
            - influxdb
          volumes:
            # Mount for kapacitor data directory
            - '/var/lib/kapacitor/:/var/lib/kapacitor'
            - '/etc/kapacitor/:/etc/kapacitor/'
            - '/etc/localtime:/etc/localtime:ro'
          ports:
            - "9092:9092"
          privileged: True
          logging:
            << : *default-logging
            options:
              tag: kapacitor
          networks:
            metrics:
              aliases:
                - kapacitor
```

### group_vars used for "all" to cover vars used in multiple groups requiring Docker, grafana, nginx, webservices, plexservers, nzbservices, telegraf
```yaml
telegraf_rpm_pkg_url: https://telegrafreleases.blob.core.windows.net/linux/telegraf-1.13.0~with~pg-1.x86_64.rpm
telegraf_deb_pkg_url: https://telegrafreleases.blob.core.windows.net/linux/telegraf_1.13.0~with~pg-1_amd64.deb


telegraf_runas_user: root
telegraf_runas_group: root
telegraf_influxdb_retention_policy: 30d
telegraf_agent_interval: 10s
telegraf_round_interval: "true"
telegraf_metric_batch_size: "1000"
telegraf_metric_buffer_limit: "10000"
percpu:
telegraf_collection_jitter: 30s
telegraf_flush_interval: 60s
telegraf_flush_jitter: 30s
telegraf_debug: False
telegraf_quiet: False
telegraf_enable_snmp: False


#### Telegraf InfluxDB Output

telegraf_influxdb_default_url: "http://{{ influxdb_server_ip }}:8086"
telegraf_influxdb_database: telegraf
telegraf_influxdb_precision: s
telegraf_influxdb_timeout: 120s
#### Telegraf TimeScaleDB Output

telegraf_timescale_connection: "host={{ telegraf_timescale_host }} user={{ telegraf_timescale_user }} password={{ telegraf_timescale_password }} sslmode={{ telegraf_timescale_sslmode }} dbname={{ telegraf_timescale_dbname }}"
#telegraf_timescale_connection: "postgres://{{ telegraf_timescale_user }}:{{ telegraf_timescale_password }}@{{ telegraf_timescale_host }}/{{ telegraf_timescale_dbname }}?sslmode={{ telegraf_timescale_sslmode }}"

telegraf_timescale_host: "10.0.10.131"

timescalepw: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          30333638316562343539363935643536313666333863343334613234326266323564353730363432
          3236653162306138376666383232303934313933636431320a656261303933303731333766336136
          66646538383939353332643435663066366264653134376137383839303235346464363036333338
          6137363938376465610a653361663032373935326365343066306230313665666530393962373462
          6538

telegraf_timescale_password: "{{ timescalepw }}"
telegraf_timescale_user: "postgres"
telegraf_timescale_sslmode: "disable"
telegraf_timescale_dbname: "timescale"
telegraf_timescale_schema: "public"
telegraf_timescale_table_template: "CREATE TABLE IF NOT EXISTS {TABLE}({COLUMNS}); SELECT create_hypertable({TABLELITERAL},'time',chunk_time_interval := '1 hour'::interval,if_not_exists := true);"
telegraf_timescale_tags_as_foreignkeys: "false"
telegraf_timescale_tags_as_jsonb: "false"
telegraf_timescale_fields_as_jsonb: "false"


#### Telegraf Output Configurations

telegraf_output_plugins:

  - name: influxdb
    options:
      urls:
        - 'http://10.0.10.176:8086'
      database: "{{ telegraf_influxdb_database }}"
      precision: "{{ telegraf_influxdb_precision }}"
      write_consistency: "{{ telegraf_influxdb_write_consistency }}"
      timeout: "{{ telegraf_influxdb_timeout }}"
  - name: postgresql
    options:
      connection: "{{ telegraf_timescale_connection }}"
      tags_as_foreignkeys: "{{ telegraf_timescale_tags_as_foreignkeys }}"
      table_template: "{{ telegraf_timescale_table_template }}"
      schema: "{{ telegraf_timescale_schema }}"
      tags_as_jsonb: "{{ telegraf_timescale_tags_as_jsonb }}"
      fields_as_jsonb: "{{ telegraf_timescale_fields_as_jsonb }

### MEDIA SERVER CONFIGS
media_root: "{{ data_mount_root }}"
media_user_uid: "{{ ssh_users.media.uid }}"
media_user_gid: "{{ ssh_users.media.gid }}"
usenet_incomplete_download_directory: /downloads/usenet/incomplete
usenet_complete_download_directory: /downloads/usenet/complete
usenet_watch_directory: /downloads/usenet/watch
torrent_incomplete_download_directory: /downloads/torrent/incomplete
torrent_complete_download_directory: /downloads/torrent/complete
torrent_watch_directory: /downloads/torrent/watch
plex_transcode_disk: False
plex_transcode_disk_mnt: /transcode
plex_transcode_disk_mode: 0774
plex_server_ip: 10.0.10.175
plex_username: devicenull
plex_password: "{{ vault_plex_password }}"
plex_token: "{{ vault_plex_token }}"


### SSH CONFIGS

sshd_permit_root_login: 'yes'

ssh_users:
 
  media:
    password: "{{ vault_media_pw }}"
    cn: "Media User"
    givenname: "Media"
    sn: "User"
    mail: "alan.janis@gmail.com"
    gecos: media
    uid: 2000
    gid: 2000
    pubkey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC29plb9A6fagU+g+zFX6Bo6SUGj58Z/6nxngV/kd95J0UJc8c3zApz6Q3s6gO1+86rpH\
    oZOKauUnXb/Kcg41ncpluDsLxu1gChY8D8jsAPxG18GxlJuB8d5rgFdX9VnCWNg4w9VuYvwv3n/7hoELyU/oC68to3VIIB4yLMGhuGY3JOqwmpckl1/\
    NhQZCeP+BTXyEib2GjJ4jqbpKXuId8/gp3UpHQ37enLF8eLHH7ue4+TxL1F1uSw3phkqLvhRWPdsYTP4ttfcReqOLpJet87yUiO0xKsUryBuCAngeSU\
    P4hYGJT8i3RbGSvyoXoLJEGIt7hFLP7dI78WPlOUIHHXNml62UKF3c8mFww4qRB95RRWTrILSftkjlHqFSIVxXNtS+AMggc60HT8L8zi5BhHg66PHAo\
    /4JWLmnfgfdMbEqpWxxa21RhTv0LySXs1IOWspXAseNWrxBCrmdxPudMPXSG6f4Gc7STs72MTWQ/0ti6rOsgMvspLshM+wWJ6L0dLRUbti+hHBcdja/\
    6J5MrOMUzgzT+VLmJG4cNP2dAfqseW6t92B9RGP1rICja5PTrIvy0kh6lyVkxnkWlC10WJo2m+VyifPjIPBkekJDfKic9VKe05glMWl8lUillQ0Y9ke\
    jFzoLXX8rfbdTmCJYg3qXTrZ4lsZQjr8dGSI4HMqw== ajanis@wwt"
    state: present
 

ssh_groups:
  media:
    description: Media Services Group
    gid: 2000
    members:
      - media
```
### Additional group_vars needed for webservices group
```yaml
nginx_accesslog_grokpattern: '%{CLIENT:client_ip} - %{NOTSPACE:ident} \[%{HTTPDATE:ts:ts-httpd}\] %{NOTSPACE:request_host:tag} "(?:%{WORD:verb:tag} %{NOTSPACE:request} (?:HTTP/%{NUMBER:http_version:float})?|%{DATA})" %{NUMBER:resp_code:tag} (?:%{NUMBER:resp_bytes:int}|-) "%{NOTSPACE:referrer}" "%{DATA:agent:tag}" (?:%{NUMBER:request_time}|-) (?:%{NUMBER:upstream_connect_time}|-) %{NOTSPACE:x_forwarded_for} %{NOTSPACE:upstream_cache_status:tag}'
fail2ban_banlog_grokpattern: '%{TIMESTAMP_ISO8601:timestamp} %{WORD:log_src}.%{WORD:src_action} *\[%{INT:fail2ban_digit}\]: %{WORD:loglevel:tag} *\[%{NOTSPACE:service:tag}\] %{GREEDYDATA:ban_status:tag} %{IP:clientip:tag}'
nginx_proxy_cache_regex: '~* (^/(photo|media|image|images|mediacover|pms_image_proxy)|\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc|css|js)$)'

nginx_index:
  - 'index.php'
  - 'index.html'
  - 'index.htm'
  - 'index.m3u8'

nginx_module_pkgs:
  - libnginx-mod-http-fancyindex
  - libnginx-mod-rtmp

nginx_modules_enabled:
  - ngx_http_fancyindex_module.so
  - ngx_rtmp_module.so


nginx_backends:

  - service: sabnzbd
    servers:
      - 10.0.10.177:8181

  - service: ombi
    servers:
      - 10.0.10.177:3579

  - service: sonarr
    servers:
      - 10.0.10.177:8180

  - service: radarr
    servers:
      - 10.0.10.177:8183

  - service: grafana
    servers:
      - 10.0.10.176:3000

  - service: tautulli
    servers:
      - 10.0.10.175:8181

  - service: bazarr
    servers:
      - 10.0.10.177:6767

  - service: plex
    servers:
      - 10.0.10.175:32400


nginx_vhosts:

# Internally accessible NON-SSL vhosts

  - servername: "home.prettybaked.com"
    serveralias: "prettybaked.com www.home.prettybaked.com {{ ansible_eth0.ipv4.address }}"
    serverlisten: "80 default_server"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";

  - servername: aquarium.home.prettybaked.com
    serverlisten: 80
    locations:
      - name: /
        docroot: /data/public_html/aquarium
        proxy: False
        extra_parameters: |
          types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
            }
          add_header Cache-Control no-cache;


  - servername: stats.home.prettybaked.com
    serveralias: stats
    serverlisten: "80"
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True

  - servername: sabnzbd.home.prettybaked.com
    serveralias: sabnzbd
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: sabnzbd
        custom_css: sabnzbd_dark.css

  - servername: sonarr.home.prettybaked.com
    serveralias: sonarr
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: sonarr
        custom_css: sonarr/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: sonarr

  - servername: radarr.home.prettybaked.com
    serveralias: radarr
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: radarr
        proxy_cache: True
        custom_css: radarr/dark.css
        extra_parameters: |
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection $http_connection;
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: radarr

  - servername: bazarr.home.prettybaked.com
    serveralias: bazarr
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: bazarr
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: bazarr
        proxy_cache: True

  - servername: ombi.home.prettybaked.com
    serveralias: ombi
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: tautulli.home.prettybaked.com
    serveralias: tautulli
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: tautulli
        proxy_cache: False
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: plex.home.prettybaked.com
    serveralias: plex
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex
ff

# Externally accessible NON-SSL vhosts

  - servername: "home.prettybaked.com"
    serveralias: "prettybaked.com www.home.prettybaked.com www.prettybaked.com {{ ansible_eth0.ipv4.address }}"
    serverlisten: "8080 default_server"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";


  - servername: ombi.prettybaked.com
    serveralias: ombi
    serverlisten: "8080"
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: tautulli.prettybaked.com
    serveralias: tautulli
    serverlisten: 8080
    locations:
      - name: /
        proxy: True
        backend: tautulli
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: plex.prettybaked.com
    serveralias: plex
    serverlisten: "8080"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex

  - servername: stats.prettybaked.com
    serveralias: stats
    serverlisten: 8080
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True

  - servername: aquarium.prettybaked.com
    serveralias: aquarium
    serverlisten: 8080
    locations:
      - name: /
        docroot: /data/public_html/aquarium
        proxy: False
        extra_parameters: |
          types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
            }
          add_header Cache-Control no-cache;


nginx_vhosts_ssl:

# Internally accessible SSL vhosts

  - servername: "home.prettybaked.com"
    serveralias: "prettybaked.com www.prettybaked.com www.home.prettybaked.com"
    serverlisten: "443 default_server"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";

  - servername: stats.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True


  - servername: sabnzbd.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: sabnzbd
        custom_css: sabnzbd_dark.css

  - servername: sonarr.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: sonarr
        custom_css: sonarr/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: sonarr

  - servername: radarr.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: radarr
        custom_css: radarr/dark.css
        extra_parameters: |
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection $http_connection;
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: radarr

  - servername: bazarr.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: bazarr
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: bazarr
        proxy_cache: True

  - servername: ombi.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: tautulli.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: False
        backend: tautulli
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: plex.home.prettybaked.com
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex

# Externally accessible SSL vhosts

  - servername: "home.prettybaked.com"
    serveralias: "prettybaked.com www.prettybaked.com www.home.prettybaked.com"
    serverlisten: "8443 default_server"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";

  - servername: ombi.prettybaked.com
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: tautulli.prettybaked.com
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: False
        backend: tautulli
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: plex.prettybaked.com
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex

  - servername: stats.prettybaked.com
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True
```
### Sample Service Role Playbook

This playbook uses additional group_vars configurations and roles to set up an nginx reverse proxy in addition to 
nbzservices and plexserver configs.

```yaml
# Deploy Plex Media Server; NZB Services (Sonarr, Radarr, SABnzbd, OMBI);
# Apache webserver (vhosts + reverse proxy configs for media services); Samba and Apple File Protocol file sharing services
# Optionally deploy with NFS or CephFS storage backends (requires openldap role)

- name: "[Media Server] :: Deploy Plex Media Server, NZB Services (SabNZBd, Sonarr, Radarr, Ombi) Monitoring Services (Tautulli, Telegraf), Web Services (Apache), File Services (Samba,AFP,Avahi) :: Includes Ansible roles for OpenLDAP-Client, CephFS/NFS"
  hosts:
    - mediaservices
    - webservices
    - plexservers
    - nzbservices
  remote_user: root
  gather_facts: yes
  vars_files:
    - vault.yml
  tasks:
    - import_role:
        name: common
      tags:
        - common
    - import_role:
        name: openldap
      when:  openldap_server_ip is defined and openldap_server_ip != None
      tags:
        - openldap
    - import_role:
        name: ceph-fs
      when:
        - shared_storage
        - storage_backend == "cephfs"
      tags:
        - cephfs
    - import_role:
        name: mediaserver
      when: "'mediaservices' in group_names"
      tags:
        - mediaservices
    - import_role:
        name: docker
      when: "'mediaservices' in group_names"
      tags:
        - docker
    - import_role:
        name: nginx
      when: "'webservices' in group_names"
      tags:
        - nginx
    - import_role:
        name: telegraf
      when: "'telegraf' in group_names"
      tags:
        - telegraf
    - setup:
      tags:
        - always

```
### systemd container service handlers
A systemd service will be created for all docker containers and compose containers
in the form of `docker-<container` or `docker-<project>-<container-name>`
```yaml
[Unit]
Description={{ item.value.description }}
# Start ocker container dependencies before starting this container
After=docker.service {% if item.value.depends_on is defined %}{% for container in item.value.depends_on %}docker-{{ container }}.service{% if not loop.last %} {% endif %}{% endfor %}{% endif %}

{% if shared_storage and (item.value.volumes | select('search', data_mount_root) | list | length) != 0 %}
# Ensure shared storage is mounted
ConditionPathIsMountPoint={{ data_mount_root }}
{% endif %}

# Require shared storage mount and docker container dependencies before starting
Requires=docker.service {% if item.value.depends_on is defined %}{% for container in item.value.depends_on %}docker-{{ container }}.service{% if not loop.last %} {% endif %}{% endfor %}{% endif %}

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a {{ item.key }}
ExecStop=/usr/bin/docker stop -t 2 {{ item.key }}
RestartSec=1min

[Install]
WantedBy=multi-user.target

```

