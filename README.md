# Sterilized manifests for some of my running services 

just adjust pvc configs and some other environments just like in docker-compose

run ```kubectl apply -f /path/to/files/. -n <yourNameSpace>```

## Table of Contents

1. [Unbound](#Unbound)
2. [ConvertX](#ConvertX)


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