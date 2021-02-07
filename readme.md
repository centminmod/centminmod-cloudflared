The below instructions for Argo Tunnel connecting to self hosted application are tailored to Centmin Mod LEMP stack users running CentOS 7 running CSF Firewall.

## Instructions For Argo Tunnel Usage For Centmin Mod LEMP Stack

Based on documentation outlined at:

* https://developers.cloudflare.com/cloudflare-one/connections/connect-apps

## Step 1. CSF Firewall Whitelisting

This is a one time task. Place in /etc/csf/csf.allow allow file whitelisting for Cloudflare route1/2 hostname's IP addresses to allow egress TCP traffic on destination port 7844. 

First command backs up csf.allow and then appends to csf.allow the CSF Firewall allow list to CF Argo Tunnel IPs for destination port 7844.

```
cp -a /etc/csf/csf.allow /etc/csf/csf.allow.backup.before-argo-tunnel
cat >> /etc/csf/csf.allow << EOF
tcp|out|d=7844|d=198.41.192.7
tcp|out|d=7844|d=198.41.192.47
tcp|out|d=7844|d=198.41.192.107
tcp|out|d=7844|d=198.41.192.167
tcp|out|d=7844|d=198.41.192.227
tcp|out|d=7844|d=198.41.200.193
tcp|out|d=7844|d=198.41.200.233
tcp|out|d=7844|d=198.41.200.13
tcp|out|d=7844|d=198.41.200.53
tcp|out|d=7844|d=198.41.200.113
EOF
```

restart CSF Firewall

```
csf -ra
```

## Step 2. Install Cloudflared Binary

```
curl -4s https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-amd64.tgz | tar xzC /usr/local/bin
cloudflared update
```

Authenticate cloudflared using your Cloudflare Account log as per https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/setup

```
cloudflared tunnel login
```

Verify cloudflared version installed

```
cloudflared --version
cloudflared version 2021.2.1 (built 2021-02-04-1528 UTC)
```

## Step 3. Create Argo Tunnel

* You can name your Argo Tunnels whatever you want, here we'll use the intended hostname/subdomain as the name of the Argo Tunnel.
* Pipe the tunnel create command output into a log file named `$hostname-tunnel-create.log` so can use the log file to find the tunnel id and credential file needed to install cloudflared as a service and assign them to variables.
* You'll need to have `jq` installed which is installed by default on Centmin Mod LEMP stacks. If not you can install it via `yum -y install jq`.

```
hostname=tun.domain.com
cloudflared tunnel create $hostname | tee $hostname-tunnel-create.log
tunnelid=$(cloudflared tunnel list -o json | jq -r --arg h $hostname '.[] | select(.name == $h) | .id')
credfile="/root/.cloudflared/${tunnelid}.json"
```

## Step 4. Creating cloudflared YAML Config File

Create the [cloudflared YAML config file](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/configuration/config) at `~/.cloudflared/config.yml` by populating the variables below

```
hostname=tun.domain.com
localhost=http://localhost:80
metrics=localhost:5432
cftag='cmm=test'
cfpid='/var/run/cmm-test-argo.pid'
cfupdatettl='24h'
```

Creating `~/.cloudflared/config.yml` file with variables you assigned

```
cat > ~/.cloudflared/config.yml <<EOF
tunnel: $tunnelid
credentials-file: $credfile
originRequest:
  connectTimeout: 30s

metrics: $metrics
#tag: $cftag
pidfile: $cfpid
autoupdate-freq: $cfupdatettl
loglevel: info
logfile: /var/log/cloudflared.log

ingress:
  - hostname: $hostname
    service: $localhost
  - service: http_status:404
EOF
```

I commented out the `tag:` setting for now as manuallying running the cloudflared argo tunnel gives an error right now

```
cloudflared tunnel --config ~/.cloudflared/config.yml run tun.domain.com
expected string slice found string for tag
```

Validate ingress rules

```
cloudflared tunnel ingress validate
Validating rules from /root/.cloudflared/config.yml
OK
```

## Step 5. Create Cloudflare CNAME DNS Record To Route Argo Tunnel

As per documentation https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/routing-to-tunnel/dns, you can create the CNAME DNS record via command line

```
cloudflared tunnel route dns <UUID or NAME> www.app.com
```

In this case

```
cloudflared tunnel route dns tun.domain.com tun.domain.com
```

or using `$tunnelid` variable you populated in step 3 above.

```
cloudflared tunnel route dns $tunnelid tun.domain.com
```

## Step 6. Install cloudflared Service on CentOS 7

Install cloudflared service, start it and ensure it starts on server reboots. You can check the cloudflared log file at `/var/log/cloudflared.log`.

```
cloudflared service install
service cloudflared start
chkconfig cloudflared on
```

Check status

```
systemctl status cloudflared.service
```

or

```
service cloudflared status
```

## Listing Argo Tunnels Created

Above method used via `cloudflared` will not list the created Argo Tunnel in Cloudflare dashboard's Traffic > Argo Tunnel section in web GUI.

To list your created Argo Tunnels use either command:

```
cloudflared tunnel list
```
or
```
cloudflared tunnel list -o json
```

## Centmin Mod Nginx Access Logs

Argo Tunnel by default [won't pass on the visitor's real IP address](https://developers.cloudflare.com/cloudflare-one/faq/tunnel#does-argo-tunnel-send-visitor-ips-to-my-origin) to Centmin Mod Nginx. Instead it will show up like

```
tail -1 /home/nginx/domains/tun.domain.com/log/access.log 
127.0.0.1 - - [07/Feb/2021:22:39:12 +0000] "GET /?test HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.104 Safari/537.36 OPR/74.0.3911.75"
```

Centmin Mod Nginx config file at `/usr/local/nginx/conf/nginx.conf` has additional Nginx log format options which can log Argo Tunnel received visitor's real IP addresses via `$http_x_forwarded_for` field.

Example for log format named `cf_custom4`

```
log_format cf_custom4 '$remote_addr - $remote_user [$time_local] $request '
              '"$status" $body_bytes_sent "$http_referer" '
              '"$http_user_agent" "$http_x_forwarded_for" "$gzip_ratio" "$brotli_ratio"'
              ' "$connection" "$connection_requests" "$request_time" $http_cf_ray '
              '$ssl_protocol $ssl_cipher $http_content_length $http_content_encoding $request_length';
```

You can alter and create your own custom log formats too.

To enable this, you need to edit Nginx vhost config files include file `/usr/local/nginx/conf/cloudflare.conf` in `/usr/local/nginx/conf/conf.d/tun.domain.com.ssl.conf` and/or `/usr/local/nginx/conf/conf.d/tun.domain.com.conf`. This enables [Cloudflare's documented restoration of real visitor IP addresses](https://support.cloudflare.com/hc/en-us/articles/200170786-Restoring-original-visitor-IPs-Logging-visitor-IP-addresses-with-mod-cloudflare-).

```
  # uncomment cloudflare.conf include if using cloudflare for
  # server and/or vhost site
  include /usr/local/nginx/conf/cloudflare.conf;
```

And also add a second access log line assigning the log format named `cf_custom4`

```
  access_log /home/nginx/domains/tun.domain.com/log/cf-ssl-access.log cf_custom4;
```

to existing one so it looks like

```
  access_log /home/nginx/domains/tun.domain.com/log/access.log combined buffer=256k flush=5m;
  error_log /home/nginx/domains/tun.domain.com/log/error.log;
  access_log /home/nginx/domains/tun.domain.com/log/cf-access.log cf_custom4;
```

Then restart Nginx

```
service nginx restart
```

Or via Centmin Mod command shortcut

```
ngxrestart
```

Visit Argo Tunnel site and then check the new access log and see the `$http_x_forwarded_for` field record the real visitor IP = `122.xxx.xxx.xxx`

```
tail -1 /home/nginx/domains/tun.domain.com/log/cf-access.log

127.0.0.1 - - [07/Feb/2021:22:39:12 +0000] GET /?test HTTP/1.1 "304" 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.104 Safari/537.36 OPR/74.0.3911.75" "122.xxx.xxx.xxx" "-" "-" "1" "1" "0.000" 61e09b05e97c32a4-MIA - - - - 1876
```