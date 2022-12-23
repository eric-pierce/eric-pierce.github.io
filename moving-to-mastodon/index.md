# Moving to Mastodon


### Why Mastodon

As part of my self-hosting journey, I shut down essentially all my social media in 2019 with the exception of my [LinkedIn](https://www.linkedin.com/in/ericrpierce/), [Spotify](https://open.spotify.com/user/p8h9h74rsfedu9e20d97kh7mj), and a twitter account which I used to follow but not to post. I didn't think it was possible to really self-host social media, but the events of the past two months and the [explosive growth](https://www.theverge.com/2022/12/20/23518325/mastodon-monthly-active-users-twitter-elon-musk) that Mastodon has undergone has changed that. Regardless of what you think about the recent changes at Twitter and the rise of Mastdon in headlines, the idea of decentralized and self-hostable social media is super interesting, and totally feasible.

{{< image src="mastomeme.jpg" caption="The Self Hoster by [Joe Dean](https://mastodon.joedean.dev/@joe/109455796397314685)" linked=false >}}

As always it's important to approach self-hosting with eyes wide open, and there are definitely pros and cons compared to the big-tech versions of social media:

{{< admonition type=success title="Pros" open=true >}}
* Your data lives on your server and you own it. You get to decide how public you want to make it. Privacy and social media are usually completely mutually exclusive, but not with Mastodon.
* No Ads, No Algorithm: Mastodon isn't a single company looking to drive revenue, so advertising and algorithm driven 
* Communities (Instances) can be local and fantastic
  * Yes this is generally the most confusing part about Mastodon, but it's also a benefit. Mastodon instances are generally built around a concept or common interest, like a location, profession, etc. I found some excellent Mastodon instances based in [Austin, TX](https://atx.pub), and several more in Chicago.
  * While facebook has groups, Instagram and Twitter don't really have a concept like this outside of hashtags and recommendations
{{< /admonition >}}

{{< admonition type=failure title="Cons" open=true >}}
* Mastodon isn't as simple for the layman as a centralized service like twitter
* Many accounts you may care about only post on twitter (though I have a workaround for this)
* Finding content or being found yourself may be difficult 
  * There is no service which searches all Mastodon or Fediverse accounts, and your Mastodon instance will only be able to search other instances it knows about.
  * If this is a big issue for you, some simple workarounds are to follow hashtags you're interested in, or leverage a relay
{{< /admonition >}}

### Setting up an Instance
Mastodon is one of several software platforms that communicate with each other through the [ActivityPub](https://en.wikipedia.org/wiki/ActivityPub) protocol. Any platform that uses ActivityPub can technically talk to each other, though some are more specialized around how closely they resemble walled garden services. Mastodon and Pleroma are two that closely resemble Twitter, and others like [Pixelfed](https://pixelfed.org/) are more closely like Instagram. Any platform which can use the ActivityPub protocol, can be classified as "part of the fediverse".

I initially tried out Plermoa due to ease of setup (it requires far less resources and work to get running), but ultimately decided to go for a full Mastodon setup. Lucky for me (any anyone who wants to set up Mastodon) the always incredible folks over at [linuxserver.io](https://www.linuxserver.io/) recently released [a single docker image](https://github.com/linuxserver/docker-mastodon) which packages up all the services needed to stand up Mastodon yourself except for redis and postgres. 

Because social media is inherently "social" I set up a new VPS and used a new domain for my Mastodon instance rather than the domain/VPS I use for my [personal services](https://eric-pierce.com/building-a-personal-cloud/). The docker compose file can be found [here](https://github.com/eric-pierce/Mastadon-and-MC/blob/main/docker-compose.yml). Here's the relevant section for mastodon, the other key services (postgres and redis) are standard installs, check out the full docker compose to see the entire setup file.

```yaml open=true
  mastodon:
    container_name: mastodon
    image: linuxserver/mastodon:latest
    restart: always
    networks:
      - traefik
      - internal
    depends_on:
      - traefik
      - postgres
      - redis
    ports:
      - 8002:80/tcp
      - 8443:443/tcp
    volumes:
      - $DOCKERDIR/mastodon:/config
    security_opt:
      - no-new-privileges:true
    environment:
      - TZ=$TZ
      - PUID=$PUID
      - PGID=$PGID
      - LOCAL_DOMAIN=$MAST_DOMAIN
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - DB_HOST=postgres
      - DB_USER=mastodon
      - DB_NAME=mastodon
      - DB_PASS=$MAST_POSTGRES_PASSWORD
      - DB_PORT=5432
      - ES_ENABLED=false
      - SECRET_KEY_BASE=$SECRET_KEY_BASE
      - OTP_SECRET=$OTP_SECRET
      - VAPID_PRIVATE_KEY=$VAPID_PRIVATE_KEY
      - VAPID_PUBLIC_KEY=$VAPID_PUBLIC_KEY
      - SMTP_SERVER=$SMTP_HOST
      - SMTP_PORT=$SMTP_PORT
      - SMTP_LOGIN=$SMTP_USERNAME
      - SMTP_PASSWORD=$SMTP_PASSWORD
      - SMTP_FROM_ADDRESS=$SMTP_FROM
      - S3_ENABLED=true
      - S3_ENDPOINT=$S3_ENDPOINT
      - S3_HOSTNAME=$S3_HOSTNAME
      - S3_BUCKET=$S3_BUCKET
      - AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
      - S3_ALIAS_HOST=$S3_ALIAS_HOST
      #- WEB_DOMAIN=$MAST_DOMAIN #optional
      #- RAILS_SERVE_STATIC_FILES=true
      #- ES_HOST=es #optional
      #- ES_PORT=9200 #optional
      #- ES_USER=elastic #optional
      #- ES_PASS=elastic #optional
    labels:
      - "traefik.enable=true"
      ## HTTP Routers
      - "traefik.http.routers.mastodon-rtr.entrypoints=https"
      - "traefik.http.routers.mastodon-rtr.rule=Host(`$MAST_DOMAIN`)"
      - "traefik.http.routers.mastodon-rtr.tls=true"
      ## Middlewares
      - "traefik.http.routers.mastodon-rtr.middlewares=chain-no-auth@file"
      ## HTTP Services
      - "traefik.http.routers.mastodon-rtr.service=mastodon-svc"
      - "traefik.http.services.mastodon-svc.loadbalancer.server.port=443"
      - "traefik.http.services.mastodon-svc.loadbalancer.server.scheme=https"
      - "traefik.http.services.mastodon-svc.loadbalancer.serversTransport=masto@file"
```

There are a few quirks to be aware of with the linuxserver.io image:
* Some services are optional such as using an S3 bucket for media hosting (I highly recommend) and implementing ElasticSearch (probably not necessary, especially for a small instance)
* Mastodon includes an internal forced 301 redirect to https, and then proxies to port 443, but it does this with a self-signed certificate. This can create challenges if you're putting the Mastodon service behind another reverse proxy like nginx, or in my case Traefik.
  * I'm actually still working with this one, but in the near term if you set your instance up to skip the internal verification of the certificate, you'll be set. All externally facing traffic will be secure, but some of the traffic inside of the container won't necessarily be. More discussion on this [here](https://github.com/linuxserver/docker-mastodon/issues/5)

### Media Storage
Social Media can generate a surprising amount of media, very quickly. Even on my single-user instance I've amassed around 12Gb of media, only a month in. One option would be to just let your server handle all the media, but that can eat up a lot of storage space, and generate high costs depending on your plan and allocated bandwidth. Backblaze put together a [fantastic guide](https://www.backblaze.com/blog/free-image-hosting-with-cloudflare-transform-rules-and-backblaze-b2/) which outlines setting up cloudflare as a caching layer in front of backblaze to bring your bandwidth costs to essentially 0, with very low storage costs and high availability of your media, all hosted under your own domain name.

### Accessing Twitter
Not all Twitter users will move to Mastodon, and there are a few ways to follow their accounts. My goal here was to have a single location to access my content, have it free of ads, and ideally to be stored on my server.

**[Nitter](https://nitter.net/)**
* Nitter is essentially an mirror of content from Twitter. I haven't personally used it, but it seems like a very simple way to access both Twitter and Mastodon content. Unfortunatley I don't see any way to bring Nitter + Mastodon into one seamless activity feed, so I explored other options.

**[BirdsiteLive](https://github.com/NicolasConstant/BirdsiteLive)**
* This is what I ended up going with. [BirdsiteLive](https://github.com/NicolasConstant/BirdsiteLive) is a bridge between the Twitter API and the ActivityPub protocol. It essentially allows you to follow any twitter account on Mastodon, just by looking up the twitter username and using the BirdsiteLive domain you set up as the "instance". 

Here's the BirdsiteLive portion of my docker-compose file:

```yaml
  bsl:
    container_name: birdsitelive
    image: nicolasconstant/birdsitelive:latest
    restart: unless-stopped
    networks:
      - traefik
      - internal
    depends_on:
      - traefik
      - postgres
    security_opt:
      - no-new-privileges:true
    ports:
      - "8009:80"
    environment:
      - TZ=$TZ
      - PUID=$PUID
      - PGID=$PGID
      - Instance:Domain=bsl.$DOMAINNAME
      - Instance:AdminEmail=name@domain.ext
      - Instance:ResolveMentionsInProfiles=true
      - Db:Type=postgres
      - Db:Host=postgres
      - Db:Name=birdsitelive
      - Db:User=$BSL_POSTGRES_USER
      - Db:Password=$BSL_POSTGRES_PASSWORD
      - Twitter:ConsumerKey=$BSL_TWITTER_API_KEY
      - Twitter:ConsumerSecret=$BSL_TWITTER_API_SECRET
      - Moderation:FollowersWhiteListing=$MAST_DOMAIN
      - Instance:Name=$BSL_NAME
    labels:
      - "traefik.enable=true"
      ## HTTP Routers
      - "traefik.http.routers.bsl-rtr.entrypoints=https"
      - "traefik.http.routers.bsl-rtr.rule=Host(`bsl.$DOMAINNAME`)"
      - "traefik.http.routers.bsl-rtr.tls=true"
      ## Middlewares
      - "traefik.http.routers.bsl-rtr.middlewares=chain-no-auth@file"
      - "traefik.http.routers.bsl-rtr.middlewares=middlewares-bsl@file"
      ## HTTP Services
      - "traefik.http.routers.bsl-rtr.service=bsl-svc"
      - "traefik.http.services.bsl-svc.loadbalancer.server.port=80"
```

There is still a fair amount to be desired with this, and it requires a special API from Twitter which needs to be "approved" (it appears that Twitter has automated this approval, and my API account request was granted immediately after I filled out the form requesting it). The API also has limits which you can see on your BirdsiteLive page on whatever domain you host it on, so you'll want to only allow instances / users to access and follow accounts (configured in the compose file).

I'd like to replace this with Nitter at some point, if there is a good solution to convert a Nitter feed into an ActivityPub feed.

### Mobile Apps

I only used twitter on my phone, so the mobile experience is pretty important. I'm an iOS user, and after evaluating a few different apps, I found two which are fantastic:
1. **Free** [Mastoot](https://apps.apple.com/gt/app/mastoot/id1501485410?l=en): This app is fairly mature, and has all the features I would want in a Mastodon app. It isn't open source, but it checks pretty much every other box.
2. **Paid** [Ivory](https://tapbots.social/@Ivory): For those who are used to Tweetbot, this will feel like home. Tapbots are working to implement a Mastodon version of Tweetbot, called Ivory. This is currently in alpha testing, so if you want to hop on the train you'll need to get in on their TestFlight list. They've been posting TestFlight openings to the Ivory Mastodon account. The alpha/beta isn't paid, but I'm sure that like Tweetbot the final product will be.

### Get Posting
Yes _Posting_ - I'm glad that the "toot" [went the way of the actual Mastodon](https://gizmodo.com/mastodon-toot-retired-twitter-tweet-equivalent-1849786221), it needed to happen for anyone to take the service seriously. 

If you have any thoughts, different experiences, or want to correct anything above, hit me up on [Mastodon](https://pierce.xyz/@eric)!
