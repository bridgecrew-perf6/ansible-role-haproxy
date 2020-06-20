# Ansible Role - HAProxy

Permet de déployer HAProxy, role forké par Elukerio.

### Requirements
- `python-apt`

### host_vars/haproxy/haproxy.yml
```yaml
base_domain: elukerio.org
haproxy_selfca_file: /etc/ssl/haproxy/elukerio-ca.pem
frontend_with_acl: user-in
backendacl:
    - name: elukerio
      sub:
      ip: ['[2001:bc8:32d7:1509::beef]','10.0.0.2']
    - name: fusion-directory
      sub: fd
      ip: ['10.0.0.1','10.0.0.2']
    - name: webmail
      sub: mail
      domain: libmail.eu
      ip: ['10.0.0.51','10.0.0.82']

haproxy_frontend:
  - name: all-in
    mode: tcp
    bind:
      - listen: ':::80'
        param:
          - v4v6
      - listen: ':::443'
        param:
          - v4v6
    tcp_request_inspect_delay:
      - timeout: 5s
    tcp_request_content:
      - action: accept
        cond: 'if { req_ssl_hello_type 1 }'
    use_backend:
      - name: admin-in
        cond: 'if { req_ssl_sni -i pve.sessionkrkn.fr }'
      - name: admin-in
        cond: 'if { req_ssl_sni -i opn.krhacken.org }'
    default_backend: user-in

  - name: user-in
    mode: http
    option:
      - forwardfor
    bind:
      - listen: 'abns@haproxy-user'
        param:
          - accept-proxy
          - ssl
          - no-sslv3
          - crt /etc/ssl/letsencrypt
    redirect:
      - string: 'scheme https code 301'
        cond: 'if !{ ssl_fc }'
    reqadd:
      - string: 'X-Forwarded-Proto:\ https'
        cond: 'if { ssl_fc }'
    rspadd:
      - string: 'Strict-Transport-Security:\ max-age=63072000'
    acl:
      - string: 'www hdr_beg(host) -i www.'
      - string: 'letsencrypt path_beg /.well-known/acme-challenge'
    reqirep:
      - search: '^Host:\ www.(.*)$ Host:\'
        string: '\1'
        cond: 'if www'
    use_backend:
      - name: letsencrypt
        cond: 'if letsencrypt'
    default_backend: drop-http

  - name: admin-in
    mode: http
    option:
      - forwardfor
    bind:
      - listen: 'abns@haproxy-admin'
        param:
          - accept-proxy
          - ssl
          - no-sslv3
          - crt /etc/ssl/letsencrypt
    redirect:
      - string: 'scheme https code 301'
        cond: 'if !{ ssl_fc }'
    reqadd:
      - string: 'X-Forwarded-Proto:\ https'
        cond: 'if { ssl_fc }'
    rspadd:
      - string: 'Strict-Transport-Security:\ max-age=63072000'
    acl:
      - string: 'pve hdr_end(host) pve.sessionkrkn.fr'
      - string: 'opn hdr_end(host) opn.sessionkrkn.fr'
    use_backend:
      - name: 'pve-interface'
        cond: 'if pve'
      - name: 'opn-interface'
        cond: 'if opn'
    default_backend: drop-http

haproxy_backend:
  - name: admin-in
    mode: tcp
    server:
      - name: admin-in
        listen: 'abns@haproxy-admin'
        param:
          - 'send-proxy-v2'
  - name: user-in
    mode: tcp
    server:
      - name: user-in
        listen: 'abns@haproxy-user'
        param:
          - 'send-proxy-v2'
  - name: letsencrypt
    mode: http
    http_request:
      - action: 'set-header'
        param: Host letsencrypt.requests
    server:
      - name: 'letsencrypt'
        listen: '127.0.0.1:8164'
  - name: pve-interface
    mode: http
    balance: roundrobin
    server:
      - name: janus
        listen: '10.0.0.1:8006'
        param:
          - check
          - ssl verify none
  - name: opn-interface
    mode: http
    balance: roundrobin
    server:
      - name: janus
        listen: '10.0.0.252:443'
        param:
          - check
          - ssl verify none
  - name: drop-http
    mode: http
    http-request:
      - action: deny
```

### License

GPLv3 - Elukerio

### Authors Informations

Mischa ter Smitten (based on work of [FloeDesignTechnologies](https://github.com/FloeDesignTechnologies))

Forked by Elukerio in June 2020
