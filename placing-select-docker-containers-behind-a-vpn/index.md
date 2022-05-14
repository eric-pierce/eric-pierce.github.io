# Placing Select Docker Containers Behind a VPN

### Who Needs A VPN?

Docker enables containers to quickly be spun up and used anywhere, but one of the risks many self-hosters take is exposing sensitive services to the open web. There is a balance between safety and convenience with each application exposed, but there are some applications which have no need to be publicly accessible. 

These typically include administrative applications such as:
* Portainer
* PGAdmin (Postgres)
* phpMyAdmin (Mariadb)
* Redis Commander
* Any other service you don't want exposed

One way to keep these services running but prevent them from being exposed is to place them behind a secure VPN. There are just a handful of major VPN protocols out there today, OpenVPN, IPSec, and the new Wireguard protocol are the big names. While Wireguard is new on the block and may not have had the level of auditing that OpenVPN has, it is blazing fast, and apparently so elegantly written that the creator of Linux, Linus Torvolds, [called it a work of art](https://lists.openwall.net/netdev/2018/08/02/124) before including it directly in the Linux Kernel.

Some may want to hide all their applications behind a VPN for very high security, while others like may only want to hide the ones which are rarely needed. This guide outlines how to configure specific services to only be accessible from behind your VPN while leaving others publicly accessible. 

### Set up a local docker-compose network

Add a new network to docker-compose for your services to live on/

```yaml
networks:
  vpn-subnet:
    external:
      name: vpn-subnet
```

### Setting up Wireguard

Although Wireguard doesn't maintain a docker container, the excellent folks at linuxserver.io do. Setting up a wireguard container in Docker is as simple as adding a new service to your docker-compose file. This docker-compose snippet assumes you have a few environment variables configured:

* **DOCKERDIR** set to the directory you want your services to store their non-temporary data files in. For this example we assume it is ~/docker/
* **DOMAINNAME** set to the domain you're hosting the content on.
* **PUID** set to the user id your docker runs under (run `id yourdockerusername` and find the numeric value associated with "uid")
* **PGID** set to the group id your docker runs under (run `id yourdockerusername` and find the numeric value associated with "gid")

```yaml
  wireguard:
    container_name: wireguard
    image: linuxserver/wireguard:latest
    restart: unless-stopped
    networks:
      vpn-subnet:
        ipv4_address: 20.20.10.5
    volumes:
      - $DOCKERDIR/wireguard:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    security_opt:
      - no-new-privileges:true
    environment:
      - TZ=$TZ
      - PUID=$PUID
      - PGID=$PGID
      - SERVERURL=wireguard.$DOMAINNAME
      - SERVERPORT=51820
      - PEERS=2
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.14.0
```

Most of the above should look pretty familiar if you've used docker-compose before, with the exception of the "sysctls" and "cap_add" sections. The "sysctls" section enables the wireguard container to manage ipv4 network connections, something we need for our internal VPN subnet. The "cap_add" section essentially grants some extra privileges to our container, such as the ability to control network functions, and manage kernel modules. We're also mounting /lib/modules as a volume into the container.

I chose to set the container specific environment variable "PEERS" to 2, which means that the container will automatically generate two profiles for connections. I personally use one for my phone to connect on the go and one for my desktop. You'll see the QR codes for these peers in the logs, and if you want to see one of the codes you can run `sudo docker exec -it wireguard /app/show-peer 1`.

### Set up DNS, Firewall Passthrough, and Port Forwarding

If you aren't using wildcard cname values for subdomains, you'll need to manually add 'wireguard' as a new CNAME entry on your DNS manager. Note that this does open up the standard wireguard port of 51820 to the internet, so if you're hosting at home you will need to forward whichever port you decide to use from your router to your server.

You'll also need to allow access to the port via your firewall

```zsh
sudo ufw allow 51820/udp 
```

### Place Container Behind your New VPN

We need to assign static ip addresses on the network we created above. The YAML format for this can be a bit different, and you may need to specify "null" as the target for any existing networks assigned to the containers. In my case I have an "internal" network used for services such as my database containers as can be seen in the examples of some services I've placed behind my VPN below:

```yaml
  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    restart: unless-stopped
    networks:
      internal: null
      vpn-subnet:
        ipv4_address: 20.20.10.3
    depends_on:
      - postgres
    security_opt:
      - no-new-privileges:true
    volumes:
      - $DOCKERDIR/pgadmin:/var/lib/pgadmin
    environment:
      - TZ=$TZ
      - PUID=$PUID
      - PGID=$PGID
      - PGADMIN_DEFAULT_EMAIL=$PERSONAL_EMAIL
      - PGADMIN_DEFAULT_PASSWORD=$PGADMIN_DEFAULT_PASSWORD
      - PGADMIN_LISTEN_PORT=5050

  pma:
    container_name: phpmyadmin
    image: phpmyadmin/phpmyadmin:latest
    container_name: phpmyadmin
    restart: unless-stopped
    networks:
      internal: null
      vpn-subnet:
        ipv4_address: 20.20.10.4
    depends_on:
      - mariadb
    secrets:
      - mysql_root_password
    environment:
      - TZ=$TZ
      - PUID=$PUID
      - PGID=$PGID
      - PMA_HOST=mariadb
      - MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql_root_password
      - ServerName=$DOMAINNAME
```

### Assign Readable Domain Names

Technically your services are accessible from the IP addresses you've assigned, after you've connected to your VPN, but who is going to remember the IP address you assigned to each? To get around this and assign standard domain names, we can utilize the COREDNS setup running in the Wireguard container.

First create a wireguard directory in your docker apps directory. This assumes that you're using your home directory as the docker apps directory

```zsh
mkdir ~/docker/wireguard/coredns/ 
```
Now we'll add the Corefile configuration it needs to manage local domain names.

```zsh
nano ~/docker/wireguard/coredns/Corefile
```

The below is an example of what the Corefile looks like for the four applications I've configured. The first portion starting with "loop" effectively forwards DNS requests to the host DNS resolver. Replace the subdomains with the applications you want to host behind your VPN. Replace EXAMPLE.COM with your domain name, no CAPS needed.
```JSON
. {
    loop
    forward . /etc/resolv.conf
}

portainer.EXAMPLE.COM:53 {
    file portainer.EXAMPLE.COM.db
    log
    errors
}

pma.EXAMPLE.COM:53 {
    file pma.EXAMPLE.COM.db
    log
    errors
}

pga.EXAMPLE.COM:53 {       
    file pga.EXAMPLE.COM.db
    log                  
    errors               
}                        

redis.EXAMPLE.COM:53 {
    file redis.EXAMPLE.COM.db
    log
    errors
}
```

Now we need to create the files referenced in the above Corefile, which hold the configuration details each service needs. Here's an example for my portainer container, again replace EXAMPLE.COM with your domain name.

```zsh
nano ~/docker/wireguard/coredns/portainer.EXAMPLE.COM.db
```

The contents should follow this structure, replacing EXAMPLE.COM with your domain name.
```
portainer.EXAMPLE.COM.        3600 IN SOA portainer.EXAMPLE.COM. YOURNAME.portainer.EXAMPLE.COM. (
                                2017042745 ; serial
                                7200       ; refresh (2 hours)
                                3600       ; retry (1 hour)
                                1209600    ; expire (2 weeks)
                                3600       ; minimum (1 hour)
                                )
portainer.EXAMPLE.COM.     IN A     20.20.10.2
```
The "serial" value can be set to any value, one suggestion would be the date you added the service.
You should also replace the "YOURNAME" in the example above with an identifier. I personally use "epierce".
You'll also want to modify the IP address on the last line of the file to be unique to that service, and to match the IP address you specified in your docker-compose file under `vpn-subnet: ipv4_address:` for that service.

### Secure Your Files

These files are a little sensitive, so let's lock them down to be root accessible only.

```zsh
sudo chown root:root ~/docker/wireguard 
sudo chmod 600 ~/docker/wireguard
```

### Testing Access

Your services are now available only when you connect to your VPN. Go ahead and connect and try accessing one of your services by going to the domain name you configured (ie pga.EXAMPLE.COM).

We aren't actually going through a reverse proxy here, and there isn't a service like Caddy, Traefik, or NGINX which manages port mapping. Services accessible over port 80 will work as normal, but services which need to be accessed on a non-standard port will need to have that port specified. For example portainer which I have configured to listen on port 9000, will need to be accessed by typing `portainer.EXAMPLE.COM:9000` in your URL bar.

Now you should be able to access your services securely, while keeping your standard services accessible as normal.
