## WAT

Boilerplate to setup a DNS server for your home usage, to resolve your own services at home.
What you want is:

 - not remembering your IPs all the time, but use `nas.myself.com`
 - have **valid SSL** certificates to access you **home** services using the browser `https://nas.myself.com`
 - No longer remember all the different web-server ports of your services, but just use `443` - so `https://nas.myself.com`
 
## Tell me more
Let's assume you have the official domain `myself.com`.
 
Now you have your own services at home and want you do want to run them using official SSL certificates.
To accomplish that, you want to utilize a "private section" of your domain for you services at home, but you do not want to 
set those settings in your primary DNS server for that domain (where ever you bught it), e.g. to not expose your network structure
or break the RFC :)

So you want to build a down DNS server for your home which does resolves

 - `gateway.myself.com` to `to 192.168.0.1` on port `443` via `https`
 - `nas.myself.com`  to `192.168.0.10` on port `5001` via `https`
 - `www.homeautomation.myself.com` to `192.168.0.6` on port `80` via `http`
 - `www.fullcustom.myself.com` to your docker service `fullcustom`
 - `www.custom1.myself.com` to your docker service `web1`

 
And if you query anything else, it recurses online to e.g. googles public DNS `8.8.8.8`. 
In addition, this will also be a DNS cache to improve your general DNS resolution at home.

Hint: This setup will allow you to still use your primary DNS server of `myself.com` for setting up public records like

 - `MX mail.myself.com google.de`
 - `A mail.myself.com 1.2.3.4`
 
And those can be resolved at home too, so it's like a `partial` overlay.

## Quick start

Prepare repo
```console
git clone https://github.com/EugenMayer/docker-dns-lb-boilerplate`
cd docker-dns-lb-boilerplate
cp .env.example .env
```

You can now start the example-based project

```console
docker-compose up -d
```

Be sure that the DNS name `www.traefik.$BASEDOMAIN` points to your machine, so it should be 127.0.0.1 in test setups.

You can now connect to your traefik admin dashboard via `http://www.traefik.$BASEDOMAIN/dashboard/` with the user `admin/admin`

Now test your setups 

```console
# assuming you have dig and you use docker-for-mac. Replace 127.0.0.1 with your docker-machine ip
# our DNS server runs on Port 55 (for testing purposes)

# our private domains
dig -p55 @127.0.0.1 nas.myself.com
dig -p55 @127.0.0.1 www.nas.myself.com

# and now a recursion
dig -p55 @127.0.0.1 google.com
```
You cannot really test the SSL-Offloading here easily without adjusting the configuration of your services, so just go on below. 

## Production start

```console
git clone https://github.com/EugenMayer/docker-dns-lb-boilerplate`
cd docker-dns-lb-boilerplate
cp .env.example .env
mkdir ../data
```

To make this production ready,
 1. edit `.env` and uncomment`FILESTORAGE`, `TRAEFIK_ACME_CASERVER`, `DNS_PORT`.
 2. Also adjust `BASEODOMAIN`, `TRAEFIK_ACME_CHALLENGE_DNS_CREDENTIALS`, `TRAEFIK_ACME_EMAIL` to your liking. For help for the value forTRAEFIK_ACME_CHALLENGE_DNS_CREDENTIALS see https://github.com/EugenMayer/docker-image-traefik#acme
 3. You now should put your `file` based rules to `data/filestorage` so `data/filestorage/nas.toml` and so on
 4. ensure `COMPOSE_FILE` is set to `docker-compose.yml`
 5. Adjust `TRAEFIK_ADMIN_AUTH_USERS` to either blank or generate a password via `docker run -it httpd htpasswd -sbn admin yourpw`


Be sure that the DNS name `www.traefik.$BASEDOMAIN` points to docker-engine machine ip.
You can now connect to your traefik admin dashboard via `http://www.traefik.$BASEDOMAIN/dashboard/` with the user `admin/admin`

See the examples in `docker-compose-test-service.yml`.

## Hands on
 
You have a preconfigure example build in here already in `docker-compose.yml`

We assume the e.g. raspberry-pi you run this stack on is on the ip `192.168.0.2`. So

- `192.168.0.2` would be your DNS setting for your router to propagate to your clients per DHCP
- `192.168.0.2` on port 80:443 also locates our central load-balancer for SSL offloading
 
#### Unbound configuration
`unbound/01_records.conf` defines the entries you want to have as private domains.
Change it to your likings, adjust the domains and IPs.

You might wonder why `www.nas.myself.com` and `nas.myself.com` is included, lets drill that down:

- `nas.myself.com` points to our NAS itself, e.g. for ssh access
- `www.nas.myself.com` will point to our central load-balancer `192.168.0.2` which does the SSL offloading to access the NAS gui

You see that for every service as a general structure. 
This means in your browser you will be using `https://www.nas.myself.com`
But for ssh you will be using `ssh root@nas.myself.com`
   

#### Load balancer / SSL offload ( Traefik )

##### Label based

For this see `docker-compose-test-service.yml`. Several examples are given.
Also see all the available labels at https://doc.traefik.io/traefik/v2.0/routing/providers/docker/

##### File based
For all your home services which do not run on your docker engine, but rather as external services, you use the file based configuration.

Let's pick the nas example at `traefik-file-examples/nas.toml`

It tells Traefik to forward every request with the host-header `www.nas.myself.com` to the backend `https://nas.myself.com:5001` which is our NAS Server-Web-Browser interface. Ensure you understart that
- `https://` means that the port 5001 needs to talk ssl/tls - if that is not the case, use `http://` - this is how to define if upstream should be ssl or not

See the [Traefik documentation](https://docs.traefik.io/configuration/backends/file/) for more about those rules if you like  
**and that's it** 


## What you have now?

1. Your Traefik ssl-offloader will get a **valid/official SSL certificates** for every `rule = "Host:www.nas.myself.com"` you define and install it.
1. When you type `https://www.nas.myself.com` into your browser, you will connect to your web-service - without any warnings
1. E.g. You can connect to your nas ssh shell using the DNS record and not remembering the ip .. `ssh root@nas.myself.com`

You never have to accept self signed certificates again, define exceptions in mobile apps to accept those at home and so on.
Wasn't that easy?

## What do you need to adjust for your setup
1. Copy this repo to your docker-engine location.
1. change `unbound/a-records.conf` to your likings and your actual `domain`, add services, fix the IP's and so on
2. adjust the rules in `traefik` to match your domain and backend ports and add as many as you like
3. Replace `TRAEFIK_ACME_CHALLENGE_DNS_PROVIDER` in `./.env` with the [DNS cloud provider](https://doc.traefik.io/traefik/https/acme/#providers) you use. I like cloudflare because its free and is supported by the Let's Encrypt clients ( its API ) - thus it offers us the ability to use [DNS-01 callenge](https://www.eff.org/de/deeplinks/2018/02/technical-deep-dive-securing-automation-acme-dns-challenge-validation) for free. Have your choice :)
4. Replace `TRAEFIK_ACME_CHALLENGE_DNS_CREDENTIALS` in `./.env` with credentials for your cloud provider with the key/values you find in the [Treafik ACME documentation](https://docs.traefik.io/configuration/acme/#provider) and put them there like it is [defined here](https://github.com/EugenMayer/docker-image-traefik#acme) (concat with `;` basically)
5. Remove `TRAEFIK_ACME_CASERVER` from the `docker-compose.yml` to disable the Let's Encrypt staging mode and you are set to go
6. Replace `DNS_PORT` in `./.env`port from `55` to `53` for the default dns port
7. Tell your gateway/router to use the ip of your docker-engine, here `192.168.0.2` as the first DNS server
 
## Credits

All this is done using [Unbound](https://nlnetlabs.nl/projects/unbound/about/) and [Traefik](https://traefik.io/) so give them hugs, do you?
