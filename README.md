# Using a Docker Nginx Reverse Proxy Example
## How to use jwilder/nginx-proxy to serve multiple virtual hosts from separate webserver containers

Let's say that you want to run multiple different webservers in docker containers, each serving up web pages on port 80, but for different domain names. In our docker-compose.yml files we can have the container listen on port 80 like this:

```
services:
  webserver1:
    image: php:7.2-apache
    ports:
      - "80:80"
    volumes:
      - $PWD/html:/var/www/html
```

But once you start one webserver listening on ports 80, you can't start another container that listens on the same port. If you try, you get an error saying that the port is already taken.

So in order to run multiple webservers in their own containers, both serving up content on port 80, you need to use a reverse proxy that listens on port 80 and forwards incoming requests to the correct webserver container based on the domain name. Then it passes back the response from the webserver container to the client.

The [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy) docker container is a great solution.

`nginx-proxy/docker-compose.yml`
```
version: '3'

services:

  nginx-proxy:
    image: jwilder/nginx-proxy
    network_mode: bridge
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
```

The jwilder/nginx-proxy is nice because it actualy connects to docker itself and watches for new containers. If the new container has a VIRTUAL_HOST environment variable, then it automatically updates its own configuration to listen for requests for the domain specified by container's VIRTUAL_HOST and then automatically starts to proxy requests for that domain to the container. Instead of having the webserver containers listen on port 80, you only expose port 80.

So, now we can create two webserver containers, witha VIRTUAL_HOST environment variable the defines which domain name should resolve to the container, and we expose port 80. We don't listen on port 80 because nginx is already listening on 80. But by exposing port 80, the nginx-proxy can direct requests to it.

`webserver1/docker-compose.yml`
```
version: '3'

services:
  webserver1:
    image: php:7.2-apache
    expose:
      - 80
    environment:
      VIRTUAL_HOST: web1.localhost
    volumes:
      - $PWD/html:/var/www/html
```

`webserver2/docker-compose.yml`
```
version: '3'

services:
  webserver2:
    image: php:7.2-apache
    expose:
      - 80
    environment:
      VIRTUAL_HOST: web2.localhost
    volumes:
      - $PWD/html:/var/www/html
```

To try out the example locally, edit your etc/hosts file and add web1.localhost and web2.localhost so you can test it.
```
127.0.0.1 web1.localhost web2.localhost
```

Start the nginx-proxy with `docker-compose up`

In separate consoles start webserver1 and webserver2 using `docker-compose up`

You will see the nginx-proxy detect the new containers as they start and begin to proxy requests to them.

Open up your browser and go to `web1.loclahost` and `web2.localhost` to see how the request is served by the appropriate webserver through the proxy.

![animated GIF of Example running](https://raw.githubusercontent.com/jmaxwilson/docker-nginx-proxy-example/master/docker-nginx-proxy.gif "exmaple running nginx reverse proxy with two webserver containers")
