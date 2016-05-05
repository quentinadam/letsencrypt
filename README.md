# letsencrypt

Minimal set of scripts to request and renew certificates from [Let's Encrypt](https://letsencrypt.org/) with NGINX, using the excellent ACME client [diafygi/acme-tiny](https://github.com/diafygi/acme-tiny).

## Installation

Clone the repo to your server, for example to `/var/letsencrypt/`:

```
sudo git clone https://github.com/quentinadam/letsencrypt.git /var/letsencrypt/
```

## Usage

### Generate an account key

First, you need to create an account private key that will allow to authenticate with Let's Encrypt :

```
sudo ./create-account
```

This script will generate a 4096-bit RSA account private key in `account/key.pem`.
If the private key file already exists, it will not be overwritten.

### Generate a domain key and certificate signing request

Then you need to add a new domain that you would like to validate and request a certificate for :

```
sudo ./add-domain example.com
```

This script will generate a 4096-bit RSA domain private key in `domains/example.com/key.pem`, generate a certificate signing request in `domains/example.com/csr.pem`, copy the renew script from `resources/renew` to `domains/example.com/renew` and copy and render the template NGINX configuration file from `resources/domain.conf` to `domains/example.com/domain.conf`.
If any of the files already exists, it will not be overwritten.

### Configure a HTTP server to validate challenges

To allow Let's Encrypt to validate the domain ownership you need to run a simple HTTP server on port 80, that will serve up challenges from a directory.
If you are using NGINX, you can use to following configuration file (by replacing `[path]` with the path where you installed this repository):

```
server {
	listen 80;

	location / {
		return 301 https://$host$request_uri;
	}

	location /.well-known/acme-challenge/ {
		alias [path]/challenges/;
		default_type text/plain;
		try_files $uri =404;
	}
}
```

Alternatively, you can run the below command to automatically install it:

```
sudo ./install-nginx
```

This script will copy and render the template configuration file from `resources/letsencrypt.conf` to `/etc/nginx/sites-available/letsencrypt.conf`, create symlink to the file in `/etc/nginx/sites-enable/` and reload NGINX.

### Request the domain certificate

You are now ready to request the domain certificate:

```
sudo ./domains/example.com/renew
```

This script will make sure the `challenges` directory exists and run the `resources/acme-tiny.py` python script from [diafygi/acme-tiny](https://github.com/diafygi/acme-tiny) to validate the domain and request the certificate. The requested certificate will be saved to `domains/example.com/cert.pem` and will be concatenated with the Let's Encrypt intermediate certificate and saved to `domains/example.com/chain.pem`.

### Install the certificate

You can use and modify the below NGINX configuration file (by replacing `[domain]` by the domain name and `[path]` by the path to `domains/[example.com]/`):

```
server {
	listen 443;
	
	server_name [domain];
	
	ssl on;
	ssl_certificate [path]/chain.pem;
	ssl_certificate_key [path]/key.pem;
	ssl_session_timeout 5m;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
	ssl_session_cache shared:SSL:50m;
	ssl_dhparam /etc/nginx/dhparam.pem;
	ssl_prefer_server_ciphers on;
	
	location / {
		default_type text/plain;
		return 200 'It works!';
	}
}
```

Alternatively, you can run the below command to automatically install it:

```
sudo ./domains/example.com/install-nginx
```

This script will copy the configuration file from `domains/[example.com]/domain.conf` to `/etc/nginx/sites-available/[example.com].conf`, create symlink to the file in `/etc/nginx/sites-enable/` and reload NGINX.

### Install cronjob to renew the certificate every month

Let's Encrypt certificates expire after 90 days, so you will need to run the `renew` script in time to request a new certificate. To request a new certificate on the first of each month, add the following line to crontab (by replacing `[path]` with the path where you installed this repository):

```
0 0 1 * * [path]/domains/example.com/renew
```

Alternatively you can run the command to automatically install it:

```
sudo ./domains/example.com/install-cron
```

This will install the renew script in crontab to run every first of the month, if the line does not yet exist in crontab.
