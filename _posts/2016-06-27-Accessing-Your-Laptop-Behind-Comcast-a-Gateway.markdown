---
layout: post
date:   2016-06-27 16:06:49 -0700
title: 
categories: comcast
---
Let's say you are a Comcast customer and you are developing on a laptop 
that's need to behave like it has a publically routable IP address.

Paypal, for example, might need to send invoice events to your service
by invoking a callback URL. Or Slack might need to send button events to 
your bot server by invoking a callback URL. Those are my use cases 
anyway. 

In both cases you need to give these services a URL to call. For development 
it's very convenient to run a service that implements these callback on 
your laptop. So the URL would look something like:
https://mylaptop.domain.com/myoperation

The problem is of course your laptop doesn't have an IP address. Your 
Comcast box gets an IP address through DHCP and NAT takes care of 
the rest. Your laptop and any service running on it are invisible
to the external world.

You have several options:

# Use a tunnel service like localtunnel (https://localtunnel.me/) or ngrok (https://ngrok.com/). 

The idea is:

* You start the process on your local machine passing in the port on which your service is running. It acts as a proxy for your process.
* The tunnel connects back to home base and prints out a URL that you can tell other services to use to connect to your service.  
* When an external service makes a request using the URL the tunnel will forward it on to your process.

Both are work well and are relatively easy to set up. Ngrok has
an advantage in that it has great debugging capabilities. You
can see the payloads that are being sent which is often quite
helpful.

The problem is the tunnels are transient. Whenever the tunnel
process is restarted the URL changes and you have to go change 
everywhere it's used. It's not a permanent solution.

# Dynamic DNS + Port Forwarding

The Comcast modem supports Dynamic DNS. A Dynamic DNS service creates
a domain name that you can give to external services. Whenever the IP 
address of the modem changes it, when configured properly, contacts 
the DDNS service and updates the IP address. So the domain name will
always map to the right IP address. My Comcast gateway only supports
DynDNS.

That's step one. That's how external services can get to your network.
The problem is your laptop can't get the request. That's where port
forwarding comes in. The modem can forward all traffic on
a certain port to an IP address and port within the network.

So you set the modem, using it's port forwarding setup, to forward 
requests to your laptop's IP address and the port of your service.

This actually works! One problem is Dynamic DNS services cost money
and the domain names are ugly.

What happens if you have your own wifi modem that connects to a 
Comcast Triple Play gateway? The secret is to set the Comcast
gateway into brigge mode. The turns the gateway into a cable modem
and removes the router functionality and the wifi functionality.
All the traffic will simply be passed directly to your modem.

Setup the modem as previously discussed. Configure its
Dynamic DNS capability and setup portfording to pass traffic
to your laptop.

# Regular DNS + Port Forwarding

If you can live without the dynamic DNS update then you can
do everything you did in #2 only instead of using a Dynamic
DNS service you use your regular DNS service.

You can find the IP address of your Comcast network  with 
a service like: http://www.myipaddress.com/show-my-ip-address/. 

Now go to your DNS service. Choose one of your many unused domain names.
Create an A record for a subdomain like mylaptop.domain.com. Set the IP 
address of the subdomain to your IP address.

Setup port forwarding as we discussed above.

Since the IP address assigned to your Comcast network doesn't change
often this approach isn't all that bad. You get a nice domain name
and it probably won't cost much because you are using your existing
DNS service.

# Becoming Your Own Proxy

In the version instead of using Nginx your process acts as a proxy. If you are
using node then there's http-proxy module that gets the job done. If 
using express you just start a http and https web server. Then the https
forwards requests on to http. 

Advantage: this approach simplifies operations in that you just have one process
to wortty about, 

Disadvantage: using a separate process offloads work from your node server. Since
node is single threaded shedding work off to different processes means your node
process can spend more time doing your work instead of the proxy work.

# Supporting SSL

If your service must support https then you can:

* Support SSL directly in your application.
* Use Nginx (or its equivalent) as a proxy.

Letsencrypt is a free service for creating certs.

The advantage of using Nginx as a proxy is that your serivce doesn't have 
to mess with SSL.

Here's an example Nginx configuration file that works a proxy. A proxy
accepts SSL requests and then forwards them to your process over HTTP.
Nginx can distribute requests to multiple backend processes, you just 
need to set the ports up correctly.

      events {
        worker_connections  1024;  ## Default: 1024
      }
    
      http {
        server {
          server_name sub.domain.com;
          listen 443 ssl;

          ssl_certificate      /cert.pem;
          ssl_certificate_key  /privkey.pem;
    
          location / {
            proxy_redirect off;
            proxy_pass http://127.0.0.1:5115;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
          }

          location ^~ /myoperation/ {
            proxy_redirect off;
            proxy_pass http://127.0.0.1:6115;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
          }
        }
      }

This could be all wrong or not the best way or stupid, but it is what I've
discovered in trying to make it work for me. Hopefully it will help you
save some work.
