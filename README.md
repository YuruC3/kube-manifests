# Sterilized manifests for some of my running services 

just adjust pvc configs and some other environments just like in docker-compose

run ```kubectl apply -f /path/to/files/. -n <yourNameSpace>```

## Table of Contents

1. [Unbound](#Unbound)
2. [ConvertX](#ConvertX)
3. [PiHole StatefullSet](#PiHole)
4. [Matrix Synapse](#MatrixSynapse)
5. [Element](#MatrixElement)
6. [Datastore](#Data)


### Unbound

Here I've set a few blocklists from [hagezi](https://github.com/hagezi/dns-blocklists). 

What you need to adjust in Deployment.yaml is these points:

1. timeZone on line 58 
2. on line 165 adjust or remove local-zone and local-data if you have/don't have local domain
3. deployment replicas on line 209

In service.yaml adjust:

1. loadBlalancerIPs on line 10 if using MetalLB

In netPolicy.yaml you just need to check if you use kube-dns as it is used for dns for pods

Also configure a cronjob to restart deployment on a dedicated server with access to controll plane, or just one of the cluster servers. 

Like so

```
# Restart every 6 hours
00 */6 * * * /usr/local/bin/kubectl -n <YourNameSpace> rollout restart deployment unbound-combo-deployment
```


### ConvertX

The only think that would need to be set is loadBalancerIP in servive.yaml. Other than that I think that it is pretty much Plug and Play. 


### PiHole

Here I would encourage to use Unbound instead because I've started getting problems with PiHole after around a month - two. Also I needed to disable "history logging". The thing that makes querry history visible in the dashboard. I've done this because it was causing a ton of reads/writes and my 1Gbps switch didn't liked it. 

nebula.yaml is for syncing between piholes

What to change: 

1. line 12 and 13 in nebula.yaml. Look at nebulas github for what to set here
2. line 128 in nebula.yaml. Set correct NS
3. lines 40 to 46 in netPolicy.yaml for access to webgui
4. lines 55 to 58 in netPolicy.yaml for Nebula syncing
5. line 50 in service.yaml
6. line 12, 74 as well as 127

Note that dnsmasq-extra-data-pvc is for storing extra data about local domains to redirect a domain and all subdomain. It works like so 

Redirect int.example.me and *.int.example.me to 4.3.2.1


### MatrixSynapse


What to change: 

1. in pvc.yaml change line 6 to you domain or remove that label.
2. in service.yaml set loadBalancerIP if you don't use reverse proxy.
3. in netPolicy.yaml set correct Postgresql databate address on line 17.
4. also in netPolicy.yaml adjust or remove lines 45 to 55



### MatrixElement

Here there is a bit of work to do to get everything working. 

First of all download the latest [element-web](https://github.com/element-hq/element-web) version and unpack it. Thereafter configure config.json. 

After doing that start temporary pod for copying over files with ``` kubectl -f copyFilesToPVCDeploy.yaml```

After it starts run ```kubectl cp ./YourElementWebFolder/ YourNameSpace/uploader-matrix-landing-page:/data/```

after that you can remove the copyFilesToPVCDeploy.yaml file and run ```kubectl apply -f /path/to/files/. -n <yourNameSpace>```



### Data

You need to change NFS share for storing shared files. Do this in deployment.yaml on line 65 and 66 

This is a combination of Nginx for serving static content like files or images, and filebrowser which is a container that has a webGUI for browsing, adding and/or removing images from a folder.

Combined together webGUI for filebrowser can be placed behind an internal-only reverse proxy with internal-only domain. Nginx on the other hand can be published on the internet.

Port 80 is used for filebrowser webGUI. Port 52345 is used for exposing Nginx to the internet.

#### Data-nginx-config

Here is how a nginx-config can look like

´´´
user nginx;
worker_processes auto;
worker_cpu_affinity auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;

events {
        worker_connections 4096;
        multi_accept on;
}

http {
        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;

        ##
        # Gzip Settings
        ##
        gzip on;

        application/javascript text/xml application/xml application/xml+rss text/javascript;


        server {
                listen 52345;
                server_name _;

                root /etc/nginx/html;


                location / {
                        index index.html;
                }

                location /assets {
                        alias /data/;
                        autoindex on;
                        autoindex_exact_size off;
                        autoindex_localtime on;

                }
        }
}
´´´

