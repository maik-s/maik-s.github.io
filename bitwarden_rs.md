# Hosting `bitwarden_rs` locally without world-wide internet access

In this guide I'd like to share the steps, that I've conducted to host the docker image `bitwarden_rs` locally and such that the container is not able to access the internet.  
The original idea for this endeavor are security considerations. First, I don't want to expose my bitwarden keestore to the public internet and second I don't want to fully trust the mentioned docker image. As a worst case scenario I think about that the docker image is kind of backdoored, such that all secret might be transfered to an attacker.

# Run the docker image

So the first step is to run the docker image in the local home network. You can do this on a raspberrypi, server, NAS or whatever you like. I use a raspberrypi with the latest raspbian buster.
Luckily, the `bitwarden_rs` image does not require lots of configuration. 
The whole `docker-compose.yml` that we use in this guide, looks like this. Dont worry, I'll will explain the different configuration options throughout the post.

```yml
version: '3.4'

volumes:
    bwdata:

services:
    bitwarden:
        container_name: bitwardenrs
        image: bitwardenrs/server:latest
        restart: always
        volumes:
            - bwdata:/data/
        ports:
             - 127.0.0.2:8080:80
        networks:
            - restricted
        dns:
            - 8.8.8.8

networks:
    restricted:
        driver_opts:
        com.docker.network.bridge.name: br_onlylocal
```

## nginx reverse proxy

You may wonder why I bind it to `127.0.0.2:8080`. This is because I run a nginx reverse proxy on the pi itself, which than hands the request to the docker container.
I do this so that I can configure a nice hostname, for the required https certificate that we will create in the following chapter.

This is the nginx config that I use:

```conf
server {
    server_name raspberrypi.local;
    listen 443 ssl;
    listen [::]:443 ssl;
    client_max_body_size        10G;
    client_body_buffer_size     400M;

    location / {
        proxy_pass http://127.0.0.2:8080;
        proxy_set_header Host   $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout   600;
        proxy_send_timeout  600;
        proxy_read_timeout  600;
        send_timeout    600;
    }

    ssl_certificate /etc/ssl/certs/raspberrypi.local.crt;
    ssl_certificate_key /etc/ssl/private/raspberrypi.local.key;
}
```

# HTTPS self-signed cert

As mentioned, this runs locally and not in the world wide web. Since bitwarden requires an encrypted connection to work, we need to create a self-signed certificate.
Creating a self-signed cert is not difficult, however you still need to consider some aspects

1. Have a handy DNS name for the host
2. depending on the clients you want to use (e.g. the iOS app), you might need to respect some requirements regarding self-signed certs. We will look into the specifics of iOS

## DNS entry

The first problem I needed to solve was getting a proper DNS name for the raspberry pi, such that I can configure it in the bitwarden clients.
I see three solutions to this problem:

1. The router provides local dns entries (which it does in my case). So, I can use the hostname (raspberrypi) + the tld ".local" (`raspberrypi.local`) to reach it from any other device within the same network.
2. If your router does not support this you can configure local dns entries in `/etc/hosts` on unix or `C:\Windows\System32\drivers\etc\hosts` on windows. However, if this I is also not possible (e.g. iOS) the third option is usefull.
3. Pointing an A DNS record of any domain to your local IP adress. As an example: Add an A record `192.168.1.14` for your `bw.local.example.org` domain at your domain registrar. But, in order to get this working you should configure your raspberrypi to have a static IP adress. Guides for this are available all over the internet.

## Creating the self-signed cert

Once you got your DNS working you can generate your self-signed cert.
I've used [this nice tutorial}(https://linuxize.com/post/creating-a-self-signed-ssl-certificate/), but had to adapt it a little, as iOS has some requirements for self-signed certs.
iOS demands that self-signed certs are not valid for longer than two years and they must include an subjectAltName entry. So I ended up with this command:

```bash
openssl req -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out raspberrypi.local.crt -keyout raspberrypi.local.key -addext "subjectAltName = DNS:raspberrypi.local"
```

I've deployed the certificates in the respective directories of `/etc/ssl` as you can see in my nginx config.

So what we already got: 
- We setup an reverse proxy with nginx with a `server_name` for our appropiate DNS entry and deployed a self-signed cert.
- A test should verify that visiting `https://raspberrypi.local` welcomes us with the bitwarden login screen.

Now we want to plug the internet cable for the docker container.

# Restrict the internet access for a certain docker container

Dockers docs [provide an example](https://docs.docker.com/network/iptables/) of how to restrict the internet access of docker containers. But following this example you restrict ALL of your containers. We only want to restrict the `bitwarden_rs` container, such that it is not possible to access the internet from the inside of the container, while other devices from the same local network are able to connect to the container.

    192.168.0.1/24   <-- -->  bitwarden_rs  -/-> internet

The following solution is very easy, even though you might think on the first look it isn't as we gonna use the powerful `iptables` tool.

What we do:

1. Add a named internet brdige for the docker container
2. Add a single `iptables` rule that restricts the internet as mentioned above.

As you can see in the docker-compose.yml from above I've added the following networking section (and include it in the `services` section):

```yml
networks:
    restricted:
        driver_opts:
            com.docker.network.bridge.name: br_onlylocal
```

This piece of code makes sure that our bitwarden container uses this networking bridge, called `br_onlylocal`.

Now we can add our iptables rule with

```bash
sudo iptables -I DOCKER-USER -i br_onlylocal ! -d 192.168.0.1/24 -j DROP
```

This rule is installed for our introduced networking interface and drops every packet, that has no destination from the IP range 192.168.0.1/24.

One more note: As you can see in the above docker-compose.yml we also configure googles DNS server `8.8.8.8`. This is because be default all docker container use the same DNS server as the host. Hence, if the host uses `192.168.0.1` as its DNS server, the container does to. This would mean, the container could still resolve DNS request, as the firewall rule lets it through. But we don't want the container to do this (DNS tunneling). Hence, we setup `8.8.8.8` (or any other IP outside our firewall rule), such that those requests get dropped.


That's all!. We now have an internet restricted self-hosted `bitwarden_rs` container.
Just don't forget to renew your cert once in a while and make sure to backup your bitwarden data storage. You don't want to loose this one.
That's it!