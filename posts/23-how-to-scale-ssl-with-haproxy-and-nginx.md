---
Title: How To Scale SSL with HAProxy and Nginx
Date: 2016-06-13
Image: https://wakatime.com/static/img/blog/load-balancing-haproxy-nginx.png
Description: Scale SSL to multiple machines using Proxy Protocol.
Author: Alan Hamlett
AuthorUrl: https://wakatime.com/@alan
AuthorGravatar: https://wakatime.com/gravatar/@alan
Category: Engineering
Tags: devops, haproxy, nginx, ssl
---

### SSL is CPU Intensive

If you haven't already enabled [SSL session caching][nginx ssl optimization], do [that NOW][ssl caching howto].
But what if you have many unique requests and your load balancer is maxing out it's CPU?
That was the case with WakaTime's load balancer, because as you use the [WakaTime plugins][editors] you are constantly making requests to our api saying you're still working on a project.
We had one load balancer terminating SSL in front of multiple app servers running our [Flask app][flask].
The Flask app servers handled the requests just fine, but the load balancer was maxing out all 16 cores negotiating SSL handshakes.

### Proxying TCP instead of HTTP

The solution is to proxy TCP instead of HTTP.
The load balancer no longer terminates SSL, it just passes the TCP connection on to your app servers unmodified.

Let's say you have two nginx app servers `10.0.0.11` and `10.0.0.12` and one haproxy load balancer `10.0.0.10`.
First, [install haproxy][install haproxy].
Then, edit `/etc/haproxy/haproxy.cfg` adding these lines:

    frontend https-in
        bind *:443
        default_backend https-servers

    backend https-servers
            mode tcp
            balance roundrobin
            server srv1 10.0.0.11:443
            server srv2 10.0.0.12:443


With an nginx config like:

    server {
        server_name  example.com;
        root /opt/example/current/app;

        listen 443 ssl http2;

        ssl_certificate /etc/ssl/example/ssl.crt;
        ssl_certificate_key /etc/ssl/example/ssl.key;

        location / {
            include uwsgi_params;
            uwsgi_pass unix:/tmp/app.sock;
        }
    }

This tells haproxy to setup a Layer 4 proxy to forward all TCP connections unmodified to the two nginx servers using roundrobin to balance the connections.
The nginx app servers will share the load of negotiating SSL and parsing the HTTP requests.

<div class="center-xs"><img src="https://wakatime.com/static/img/blog/load-balancing-haproxy-nginx-small.png" class="img-responsive img-thumbnail m-bottom-xs-20" style="width:50%" /></div>

**One catch though**, your nginx app servers will see the requests coming from the IP address of your haproxy load balancer instead of the originating client.
To fix this, enable Proxy Protocol to forward the originating client's IP address to your nginx app servers.

### Forwarding the User's Real IP using Proxy Protocol

[Proxy Protocol][proxy protocol] forwards the originating client's IP address from haproxy to nginx without having to modify the HTTP request headers.
To enable Proxy Protocol in haproxy, add the `send-proxy` keyword to your `/etc/haproxy/haproxy.cfg` file:

    frontend https-in
        bind *:443
        default_backend https-servers

    backend https-servers
            mode tcp
            balance roundrobin
            server srv1 10.0.0.11:443 send-proxy
            server srv2 10.0.0.12:443 send-proxy

And configure nginx to receive the client's real IP forwarded with Proxy Protocol:

    server {
        server_name  example.com;
        root /opt/example/current/app;

        listen 443 ssl http2 proxy_protocol;

        set_real_ip_from 10.0.0.10/32;
        real_ip_header proxy_protocol;

        ssl_certificate /etc/ssl/example/ssl.crt;
        ssl_certificate_key /etc/ssl/example/ssl.key;

        location / {
            include uwsgi_params;
            uwsgi_pass unix:/tmp/app.sock;
        }
    }

Notice how we told nginx to trust the IP address of your haproxy load balancer `10.0.0.10` to give us the client's real IP.
SSL is distributed among your two nginx app servers, and your nginx log files show the correct client IP address for each request.

Now you can scale to infinity!\*

\* [Increase your file descriptors limit][file descriptors limit] so haproxy and nginx can handle more simultaneous connections.

[nginx ssl optimization]: https://nginx.org/en/docs/http/configuring_https_servers.html#optimization
[ssl caching howto]: https://bjornjohansen.no/optimizing-https-nginx
[editors]: https://wakatime.com/editors
[flask]: https://www.palletsprojects.com/p/flask/
[install haproxy]: https://haproxy.debian.net
[proxy protocol]: https://www.haproxy.org/download/1.7/doc/proxy-protocol.txt
[file descriptors limit]: https://www.cockroachlabs.com/docs/stable/recommended-production-settings.html#increase-the-file-descriptors-limit
