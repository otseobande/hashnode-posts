## The Tale Of Middlemen: Proxies, Reverse Proxies, and Load Balancers

Every day we interact with middlemen who help make our lives easier. They could be retailers or even stockbrokers. Middlemen help with the distribution of products or as agents that perform actions on our behalf. When dealing with computers, sometimes we need to set up these kinds of relationships between two or more computers to help make communication more efficient or more secure. In this article, we'll be exploring the tale of three types of middleman server setups using Nginx as my server application of choice. The middlemen we would be discussing are proxies, reverse proxies, and load balancers.

Before we explore these middlemen, we need to understand what clients and servers mean as they are the foundations of the relationships these middlemen improve upon. I'll try to explain them as simply as possible.

### What is a Client?

A client is computer hardware or software that makes a request or requests to other hardware or software (servers).

### What is a Server?

A server is computer hardware or software that responds to requests from other computers (clients).

## The Proxy Middleman

![Proxy Server.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1592775057716/RDy2VD71W.png)

A proxy server acts as the middleman between a client and multiple servers. It helps hide the identity of the client computer which could be from your internet browser by masking the IP address of the request as it's own. This is very useful for bypassing internet browsing restrictions usually put in place by companies, countries, or even websites like Netflix. Let's discuss an example, If Ade wants to watch a movie that can only be accessed from America and he stays in Nigeria, he can set up a proxy server hosted in America and tunnel all his browsing traffic through that. To Netflix, Ade is browsing from America but he really is in Nigeria. This is similar to what VPN does but the main difference is VPNs provide encryption. You can read more about the differences between VPNs and Proxy servers [here](https://www.cbtnuggets.com/blog/technology/networking/vpn-v-proxy-whats-the-difference). Proxy servers are usually used for their security and privacy benefits but they might result in a slower internet experience as requests have to pass through the middleman which might also be in use by other clients. 
Proxy servers are also referred to as forward proxies because of its nature of forwarding requests

Let's take a look at how we would implement a simple Proxy server using Nginx.

```
server {
    listen  80;
    server_name www.domain.com;

    location / {
        proxy_pass $scheme://$http_host$uri$is_args$args;
    }
}
```

I think it's important to note that this is a simple configuration but in real setups, other edge cases might need to be covered. Let's take a quick look at the code above. The server listens on port 80 for HTTP requests and proxies the requests to the host specified by the request. This creates the illusion to the target server that the request came from this server.

## The Reverse Proxy Middleman

![Reverse Proxy Server.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1592775149849/M8b1IPgcr.png)

Unlike a Proxy server that tunnels requests from a particular client to other servers. The Reverse Proxy is a Middleman that sits in front of the server and prevents requests from clients from reaching the server directly. Reverse proxies are used to protect the servers from exploits like Distributed Denial of Service (DDoS). They are also used for SSL encryption and decryption which reduces the computational load on the main server. Nginx Reverse Proxies can also be used as an API Gateways for applications with a microservice architecture as it forwards requests from various clients to the right servers by matching the routes. I have also used reverse proxy servers to configure public access to servers that whitelists just specific IP addresses from accessing them. In these cases, the trick is to whitelist just the IP address of the reverse proxy server and then make requests to the reverse proxy server instead of the main server and everything works smoothly. 

Let's take a look at how creating a reverse proxy server to google.com can be done with Nginx

```
server {
    listen 80;
    server_name www.domain.com;

    location / {
      proxy_pass http://www.google.com;

      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_redirect off;
    }
}
```

This setup is similar to the proxy setup with a small difference, the `proxy_pass` value is specified and doesn't rely on request variables. This server listens to HTTP requests and forwards them to www.google.com. The `proxy_set_header` config options are used here to let the server know the real IP address of the client. This can be helpful if the server needs to know otherwise it should work fine without it.

## The Load Balancer Middleware


![Load Balancer.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1592775256752/wM88bQusD.png)

The load balancer can be thought of as a reverse proxy that sends the client's requests to multiple servers. Load balancers help distribute traffic coming from multiple clients to multiple servers to increase latency and help provide server redundancy to help improve the reliability of the application being served. Normal cases involve balancing traffic to multiple servers serving the same version of an application using different load balancing/distribution strategies.  Learn more about load balancing strategies [here](https://kemptechnologies.com/load-balancer/load-balancing-algorithms-techniques/).

Let's take a look at an Nginx load balancer that distributes traffic to three servers 1.1.1.1, 1.1.1.2, and 1.1.1.3.

```
upstream myproject {
    server 1.1.1.1;
    server 1.1.1.2;
    server 1.1.1.3;
}

server {
    listen 80;
    server_name www.domain.com;

    location / {
      proxy_pass http://myproject;
    }
}
```

Nginx uses `upstream` to handle the list of servers to load balance to. This is a simple setup real applications might require more configurations but this should be fine for a simple use case. It's also possible to configure an Nginx server as an API Gateway that load balances several instances of each service in a microservices setup.

## Conclusion

Proxies or Middlemen as I like to call them, play important roles on the internet we use today and we interact with them when we visit our favorite websites and apps. I hope this article was able to explain how they work clearly. Please feel free to discuss this with me in the comments.

### References
- [Client-Server Model - Wikipedia](https://en.wikipedia.org/wiki/Client%E2%80%93server_model)
- [Using Nginx as a forward proxy - ef.gy](https://ef.gy/using-nginx-as-a-proxy-server)
- [What is a Reverse Proxy - CloudFlare](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/)
- [Load Balancing Algorithms and Techniques - Kemp Technologies](https://kemptechnologies.com/load-balancer/load-balancing-algorithms-techniques/)
- [Simple Load Balancing - nginx.com](https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample/)