## WAT

Boilerplate to setup a DNS server for your home usage, to resolve your own domains at home.

Let's assume you have the official domain `myself.com`.
 
Now you have your own services at home and want you do want to run them using official SSL certificates.
To accomplish that, you want to utilize a "private section" of your domain for you services at home, but you do not want to 
set those settings in your primary DNS server for that domain (where ever you bught it), e.g. to not expose your network structure
or break the RFC :)

So you want to build a down DNS server for your home which does resolve

 - `gateway.myself.com` `to 192.168.0.1`
 - `nas.myself.com` to `192.168.0.10`
 - `homeautomation.myself.com` to `192.168.0.6`
 
And if you query anything else, it recurses online to e.g. googles public DNS `8.8.8.8`. 
In addition, this will also be a DNS cache to improve your general DNS resolution at home.

Hint: This setup will allow you to still use your primary DNS server of `myself.com` for setting up public records like

 - `MX mail.myself.com google.de`
 - `A mail.myself.com 1.2.3.4`
 
And those can be resolved at home too, so it's like a `partial` overlay.

## Quick start

```bash
docker-compose up
# assuming you have dig and you use docker-for-mac. Replace 127.0.0.1 with your docker-machine ip
dig @127.0.0.1 nas.myself.com
dig @127.0.0.1 www.nas.myself.com
```

You cannot really test the SSL-Offloading here easily without adjusting the configuration of your services, so just go on below. 


## Hands on
 
You have a preconfigure example build in here already in `docker-compose.yml`

We assume the e.g. raspberry-pi you run this stack on is on the ip `192.168.0.2`. So

- `192.168.0.2` would be your DNS setting for your router to propagate to your clients per DHCP
- `192.168.0.2` on port 80:443 also locates our central load-balancer for SSL offloading
 
#### Unbound configuration
`unbound/a-records.conf` defines the entries you want to have as private domains.
Change it to your likings, adjust the domains and IPs.

You might wonder why `www.nas.myself.com` and `nas.myself.com` is included, lets drill that down:

- `nas.myself.com` points to our NAS itself, e.g. for ssh access
- `www.nas.myself.com` will point to our central load-balancer `192.168.0.2` which does the SSL offloading to access the NAS gui

You see that for every service as a general structure. 
This means in your browser you will be using `https://www.nas.myself.com`
But for ssh you will be using `ssh root@nas.myself.com`
   

#### Load balancer / SSL offload ( Traefik )
`traefik` are your frontend/backend rules for your load-balancer. It defines for which domain you want to loadbalance to which backend.
Let's pick the nas example at `traefik/nas.toml`

It tells Traefik to forward every request with the host-header `www.nas.mysql.com` to the backend `https://nas.myself.com:8080` which is our NAS Server-Web-Browser interface.

Same is done for our `gateway`, your router and your `homeautomation`, assuming they have other backend ports just as an example

More to how define rule can be found in the [Traefik documentation](https://docs.traefik.io/configuration/backends/file/) at  
**and that's it** 


## What you have now?

1. Your Traefik ssl-offloader will get proper SSL certificates for every `rule = "Host:www.nas.myself.com"` you define and install it.
1. When you type `https://www.nas.myself.com` in your browser, you will have a valid SSL certificate while accessing your NAS
1. You can connect to your nas ssh using `ssh root@nas.myself.com`

Never have to accept self signed certificates again, define exceptions in mobile apps to accept those at home and so on.
Wasn't that easy?

## What do you need to ajust for your setup
1. Copy this repo to your docker-engine location.
1. change `unbound/a-records.conf` to your likings and your actual `domain`, add services, fix the IP's and so on
2. adjust the rules in `traefik` to match your domain and backend ports and add as many as you like
3. Replace `TRAEFIK_ACME_CHALLENGE_DNS_PROVIDER` in `./.env` with the [DNS cloud provider](https://docs.traefik.io/configuration/acme/#provider) you use. I like cloudflare because its free and is supported by the Let's Encrypt clients ( its API ) - thus it offers us the ability to use [DNS-01 callenge](https://www.eff.org/de/deeplinks/2018/02/technical-deep-dive-securing-automation-acme-dns-challenge-validation) for free. Have your choice :)
4. Replace `TRAEFIK_ACME_CHALLENGE_DNS_CREDENTIALS` in `./.env` with credentials for your cloud provider with the key/values you find in the [Treafik ACME documentation](https://docs.traefik.io/configuration/acme/#provider) and put them there like it is [defined here](https://github.com/EugenMayer/docker-image-traefik#acme) (concat with `;` basically)
5. Remove `TRAEFIK_ACME_CASERVER` from the `docker-compose.yml` to disable the Let's Encrypt staging mode and you are set to go
 
## Credits

All this is done using [Unbound](https://nlnetlabs.nl/projects/unbound/about/) and [Traefik](https://traefik.io/) so give them hugs, do you?