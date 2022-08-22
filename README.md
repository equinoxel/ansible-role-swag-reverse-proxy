# SWAG Reverse Proxy

This is am opinionated role to install templates in SWAG. It will install templated into the `swag_config` path. It restarts the SWAG container (declared via)

**Note**: The role installs the configuration files in `/nginx/site-confs/` rather than `/nginx/proxy_confs`, because the latter didn't work for me.

## Role Variables

This role uses the variables listed below, along with default values (see defaults/main.yml).

First, you need to specify the location of the SWAG volume and the name of the swag container (if any)

```yml
swag_container_name: "swag"
swag_data: "/mnt/data/swag"
```

The list of reverse proxies is declared under `swag_sites_config` like so:

```yml
swag_sites_config:
  - src:
      protocol: https
      host: core
      port: '8443'
    template_name: unifi.config.j2
    alias: unifi
    domain: 'example.com'
  - src:
      protocol: https
      host: services
      port: '9001'
    template_name: portainer.config.j2
    alias: portainer
    domain: 'example.com'
```

Each reverse proxy entry contains the following items:

1. `src` - defines the parameters of the source:
    - `protocol` - the protocol for accessing the internal URL (http or https). My sites are usually *http*
    - `host` - the host name as specified in your ansible *hosts.yml* file.
    - `port` - the port used by the internal site (e.g. the exposed port of the docker container)
2. `template_name` - the template name
3. `alias` - the prefix of the reverse proxy mapper. The full domain will be defined by `{{alias}}.{{domain}}`
4. `domain` - the domain suffix for the website.

The role creates the following variables available to the template:

```yaml
    site_internal_url: "{{ item.src.protocol }}://{%if hostvars[item.src.host].ansible_host is defined %}{{ hostvars[item.src.host].ansible_host }}{% else %}{{item.src.host}}{% endif %}:{{ item.src.port }}"
    template_name: "{{ item.template_name }}"
    site_name: "{{ item.alias }}.{{ item.domain }}"
    domain: "{{ item.domain }}"
```

Please note how the site internal URL is constructed using:
  - the `src.protocol`,
  - the IP address for the host specified in `item.src.host` if found, or the `item.src.host` itself if not and
  - the `src.port`.

### Templates

Templates are easy adaptations of the *static* ones found online. For example, my Unify Controller template looks like this:

```jinja2
server {
  listen 80;
  listen [::]:80;
  server_name {{ site_name }};

  return 301 https://$server_name$request_uri;
}
server {
  listen 443 ssl;
  listen [::]:443 ssl;

  include /config/nginx/ssl.conf;

  server_name {{ site_name }};

  location / {
    proxy_set_header    Host $http_host;
    proxy_set_header    X-Forwarded-Host $host;
    proxy_set_header    X-Forwarded-Server $host;
    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header    X-Real-IP $remote_addr;
    proxy_set_header    X-Scheme $scheme;
    proxy_set_header    Referer "";
    proxy_set_header    Upgrade $http_upgrade;
    proxy_set_header    Connection "upgrade";
    proxy_pass          {{ site_internal_url }};
  }
}
```

I did the following adaptations to the static template:

- Used `{{ site_name }}` for the reverse proxy name,
- Used `{{ site_internal_url }} for the *proxy_pass* parameter and
- Added `include /config/nginx/ssl.conf;` to use SWAG's SSL configuration.

You can take inspiration from SWAG's own reverse proxy configurations.

Dependencies
------------

This role is designed to be used in conjunction with `laurivan.swag`. You can however use it stand-alone if you have already installed SWAG; just point **swag_data** to the proper location.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yml
- hosts: servers
  roles:
      - laurivan.swag_reverse_proxy
```

License
-------

MIT

Author Information
------------------

This role was created in 2022 by [Laur Ivan](https://www.laurivan.com)