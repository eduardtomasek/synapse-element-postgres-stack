# Synapse - Postgres - Element stack [Sunday Projects]

Stack providing self-hosted Matrix server with client. Default Sqlite database is switched to Postgres.

## Docker images
 * portainer/portainer-ce:latest
 * matrixdotorg/synapse:latest
 * postgres:14
 * vectorim/element-web:latest

## Working setup
 * RPI 3B+
 * SSD

## Setup stack
Generate Synapse server configuration.
```bash
docker run -it --rm \
    -v "/var/docker_data/matrix:/data" \
    -e SYNAPSE_SERVER_NAME=matrix.YOUR_DOMAIN.com \
    -e SYNAPSE_REPORT_STATS=no \
    matrixdotorg/synapse:latest generate
```

Create Element config file.

```bash
sudo nano /var/docker_data/element-config.yml
```

Copy this into file and fix domain name.
```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.YOUR_DOMAIN.com",
            "server_name": "matrix.YOUR_DOMAIN.com"
        },
        "m.identity_server": {
            "base_url": "https://vector.im"
        }
    },
    "brand": "Element",
    "integrations_ui_url": "https://scalar.vector.im/",
    "integrations_rest_url": "https://scalar.vector.im/api",
    "integrations_widgets_urls": [
        "https://scalar.vector.im/_matrix/integrations/v1",
        "https://scalar.vector.im/api",
        "https://scalar-staging.vector.im/_matrix/integrations/v1",
        "https://scalar-staging.vector.im/api",
        "https://scalar-staging.riot.im/scalar/api"
    ],
    "hosting_signup_link": "https://element.io/matrix-services?utm_source=element-web&utm_medium=web",
    "bug_report_endpoint_url": "https://element.io/bugreports/submit",
    "uisi_autorageshake_app": "element-auto-uisi",
    "showLabsSettings": true,
    "piwik": {
        "url": "https://piwik.riot.im/",
        "siteId": 1,
        "policyUrl": "https://element.io/cookie-policy"
    },
    "roomDirectory": {
        "servers": [
            "matrix.org",
            "gitter.im",
            "libera.chat"
        ]
    },
    "enable_presence_by_hs_url": {
        "https://matrix.org": false,
        "https://matrix-client.matrix.org": false
    },
    "terms_and_conditions_links": [
        {
            "url": "https://element.io/privacy",
            "text": "Privacy Policy"
        },
        {
            "url": "https://element.io/cookie-policy",
            "text": "Cookie Policy"
        }
    ],
    "hostSignup": {
      "brand": "Element Home",
      "cookiePolicyUrl": "https://element.io/cookie-policy",
      "domains": [
          "matrix.org"
      ],
      "privacyPolicyUrl": "https://element.io/privacy",
      "termsOfServiceUrl": "https://element.io/terms-of-service",
      "url": "https://ems.element.io/element-home/in-app-loader"
    },
    "sentry": {
        "dsn": "https://029a0eb289f942508ae0fb17935bd8c5@sentry.matrix.org/6",
        "environment": "develop"
    },
    "posthog": {
        "projectApiKey": "phc_Jzsm6DTm6V2705zeU5dcNvQDlonOR68XvX2sh1sEOHO",
        "apiHost": "https://posthog.element.io"
    },
    "features": {
        "feature_spotlight": true
    },
    "map_style_url": "https://api.maptiler.com/maps/streets/style.json?key=fU3vlMsMn4Jb6dnEIFsx"
}
```

Go to `/var/docker_data/matrix` and edit `homeserver.yaml`.

Comment database settings for `sqlite3`.

Add settings for Postgres database.

```yaml
database:
  name: psycopg2
  args:
    user: synapse
    password: SUPERSECRETLONGPASSWORD
    database: synapse
    host: postgres
    cp_min: 5
    cp_max: 10
```

And then start containers.

```bash
docker-compose up -d
```

Use Portainer to check state of containers. Setting up database by Synapse takes some time.

When everything is ok and running without error setup first user.

Enter into synapse container.

```bash
docker exec -it synapse /bin/bash
```

Register new matrix user.

```bash
register_new_matrix_user -c /data/homeserver.yaml http://127.0.0.1:8008
```

Follow on screen prompts.

Now is everything prepared. Go to Element on your domain and login.

## Setup reverse proxy

For self-hosted Matrix server and Element can be used Nginx as reverse proxy.

Synapse Nginx config example.
```
server {
    listen 80;
    server_name matrix.YOUR_DOMAIN.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    listen 8448 ssl http2 default_server;
    listen [::]:8448 ssl http2 default_server;

    server_name matrix.YOUR_DOMAIN.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/YOUR_DOMAIN.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN.com/privkey.pem; # managed by Certbot

     location /.well-known/matrix/client {
                return 200 '{"m.server": {"base_url": "matrix.YOUR_DOMAIN.com:443"}}';
                default_type application/json;
                add_header Access-Control-Allow-Origin *;
        }

    location /.well-known/matrix/server {
                default_type application/json;
                add_header Access-Control-Allow-Origin *;
                return 200 '{"m.server":"matrix.YOUR_DOMAIN.com:443"}';
        }

    location / {
        proxy_pass http://192.168.0.123:8008;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Element Nginx config example.

```
 server {
    listen 80;
    server_name element.YOUR_DOMAIN.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name element.YOUR_DOMAIN.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/YOUR_DOMAIN.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN.com/privkey.pem; # managed by Certbot

    location / {
        proxy_pass http://192.168.0.123:80;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```