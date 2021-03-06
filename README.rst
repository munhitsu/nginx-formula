
nginx-formula
=============

nginx tuning (this can still be used - 07/10/2015)
------------

- list of parameters that can be modified via the pillar
- defaults in `nginx/map.jinja`::

        'http_core_module_config': {
            'client_max_body_size': '50k',
        },
        'core_module_config': {
            'worker_rlimit_nofile': '4096',
        },
        'main_events_context': {
          'worker_connections': '1024',
        },

- use in pillar (from the <envname>.sls file)::

        nginx:
          version: 1.4.6-1ubuntu3.2
          core_module_config:
            worker_rlimit_nofile: 10000
            worker_processes: auto
          main_events_context:
            worker_connections: 5000
          logs:
            formats:
              # NOTE: logstash_json format is always present,
              # you only need to define additional formats
              my_custom_format: 'my custom format definition'
            access_logs:
              - path: '/var/log/nginx/logstash_access.log'
                format: logstash_json
              - path: '/var/log/nginx/my_custom_access.log'
                format: my_custom_format
            error_logs:
              - path: '/var/log/nginx/error.log'
                format: error

      docker_envs:
        yoursubdomain.yourdomain.dsd.io:
          nginx_port: 80
          client_max_body_size: 20m
          ssl:
            redirect: True
          logs:
            formats:
              # NOTE: logstash_json format is always present,
              # you only need to define additional formats
              my_custom_format: 'my custom format definition'
            access_logs:
              - path: '/var/log/nginx/logstash_access.log'
                format: logstash_json
              - path: '/var/log/nginx/my_custom_access.log'
                format: my_custom_format
            error_logs:
              - path: '/var/log/nginx/error.log'
                format: error

The following information are almost certainly obsolete. Check instead the new moj-docker-deploy-formula: https://github.com/ministryofjustice/moj-docker-deploy-formula/tree/master/moj-docker-deploy/apps
-------------

A set of typical nginx configs that cover 90% use-cases.
I.e. they expect that each app comes with::

    root_dir/500.html
    root_dir/503.html
    root_dir/404.html


templates
---------
inheritance tree::

    vhost-base.conf
    |-- vhost-fpm.conf
    |-- vhost-proxy.conf
    |-- vhost-static.conf
    `-- vhost-unixproxy.conf
        `-- vhost-unicorn.conf



vhost-base.conf/vhost-static.conf
---------------------------------

required variables:
 - appslug
 - root_dir

optional variables:
 - index_doc - if defined becomes the index file (default index.html)
 - server_name - if defined than nginx listens on server_name - think vhost (default '_')
 - autoindex - to enable nginx autoindex (default False)
 - is_external - service is user facing i.e. can enter maintenance mode


vhost-fpm.conf
---------------

required:
 - appslug
 - root_dir
 - fastcgienv - a dictionary of variables to pass to fastcgi (as fastcgi_param)

optional:
 - index_doc
 - server_name
 - is_external


vhost-proxy.conf
----------------
required:
 - appslug
 - proxy_to - i.e. localhost:5151

optional:
 - root_dir - defaults to /srv/{{appslug}}
 - index_doc
 - server_name
 - is_external


vhost-unicorn.conf/vhost-unixproxy.conf
---------------------------------------
required:
 - appslug
 - root_dir
 - unix_socket

optional:
 - index_doc
 - server_name
 - is_external


example
-------
pillar::

    nginx:
      port: 80
      http_core_module_config:
        types_hash_max_size 2048
        types_hash_bucket_size 64
      core_module_config:
        worker_rlimit_nofile: <value>

    apps:
      foo:
        nginx:
            port: 80 (defaults to nginx.port)
            redirects: []
            http_access_rules: []
            external_url: http://www.example.com
            enforce_www: False
            enforce_no_www: False
            auth_basic: True
            auth_basic_users:
              my_user: my_pass
              my_admin: my_pass
            is_external: False
            client_max_body_size: None

    maintenance:
      enabled: False
      password: changeme

grains::

    provider: vagrant (defaults to ec2)


usage example
-------------
example::

    include:
      - nginx

    /etc/nginx/conf.d/foo.conf:
      file:
        - managed
        - source: salt://nginx/templates/vhost-proxy.conf
        - template: jinja
        - user: root
        - group: root
        - mode: 644
        - context:
            appslug: foo
            server_name: foo.*
            proxy_to: localhost:9876
        - watch_in:
          - service: nginx


Don't forget to manage the logs. I.e. by::

    {% from 'monitoring/logs/lib.sls' import logship2 with context %}

    {{ logship2('foo-access',  '/var/log/nginx/foo.access.json', 'nginx', ['nginx', 'foo', 'access'],  'rawjson') }}
    {{ logship2('foo-error',  '/var/log/nginx/foo.error.json', 'nginx', ['nginx', 'foo', 'error'],  'json') }}


apparmor
--------

This formula includes some simple default apparmor profiles.

You can add extra profiles for your site specific uses by putting files into
``/etc/apparmor.d/nginx_local`` and then restarting the service - you will need
to do this to add read access to web roots or SSL certificates.

App armor is by default in complain mode which means it allows the action and
logs. To make it deny actions that the profile doesn't cover set the following
pillar::

    apparmor:
      profiles:
        nginx:
          enforce: ''


maintenance mode
----------------
Nginx templates also provide a simple and standardized mechanism to enable/disable maintenance mode for the system.
It returns your 503 page with 503 http code plus it allows you to still access the site if you pass the password
anywhere in user agent header.

To swap your system into maintenance mode make sure you've specified the maintenance password in pillar.
pillar::

    mainenance:
        password: your_password

And than just update grain & run state.highstate
grains::

    maintenance: True

Maintenance mode is only enabled for external services (is_external context variable in template see above).

