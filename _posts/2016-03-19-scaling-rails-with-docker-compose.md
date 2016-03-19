---
title: Scaling Rails with Docker Swarm and Compose
layout: post
---

Horizontally scaling a Rails app at the push of a button is something that I have never quite managed to get right.
Using chef to provision new app servers, reconfigure a load balancer and then deploying code using Capistrano is about
as close as I've managed to get it. Until now.

Recently at work there's been a debate about how to automate infrastructure management, and to make deploying multiple
resilient and load balanced applications easy. The decision came down to either Docker Swarm or Kubernetes, so I
decided to see how things might look using Swarm. I'm a big fan of Rails and developing applications for it takes up the
majority of my coding time, so deploying it was the obvious way to put Swarm to the test. I decided to use Docker
Compose for orchestration because it seemed to be the easiest way of going about it.

### Setting up the cluster

After reading through the docs for Swarm and getting my head around what a Swarm cluster consists of, I followed [this
guide](https://docs.docker.com/swarm/swarm_at_scale/) and made myself a scalable (but highly insecure) Swarm cluster.
Here's a quick diagram of what my cluster looked like:

{% highlight text %}
         ------------     ------
        |Swarm master|   |Consul|
         ------------     ------

  --------      --------      ------------
 |Worker 1|    |Worker 2|    |storage node|
  --------      --------      ------------

            --------------------
           |Frontend (interlock)|
            --------------------
{% endhighlight %}

Whether or not this is an optimal setup is beyond the scope of what I'm doing here. The idea behind my layout is that
all data stores (such as Postgres, MySql etc) will be constrained the storage node, meaning we don't have to worry about
using network storage drivers for volumes. All application containers will run on any of the worker nodes which can be
added and removed as needed, and then the frontend node is running [Interlock v1.0.0](https://github.com/ehazlett/interlock)
along with HAProxy exposed on port 80.

### Configuring the Rails application

With the swarm cluster up and running the next job is to configure our Rails app in preparation for deploying to the
Swarm. The idea here to to make the app completely isolated, so you can run any number of them without worrying about
them bumping into each other, and having no external dependencies which can prevent the container from starting up.
The [12 factor app](http://12factor.net/) methodology is key here and if you follow all of the steps on there you should
be ready to go in no time. Specific to Rails, there are just a few things you need to take into account.

* All database config should come in the form of environment variables
* All environment specific config should come in the form of environment variables
* Static assets should be served without the need for a reverse proxy
* Logging should all be done to STDOUT so Docker can control where it goes

The [rails_12factor](https://github.com/heroku/rails_12factor) gem takes care of two of these concerns for you, so once
you've sorted this the last thing to do is create your Dockerfile. This will be a very minimal file, as all it has to do
is install your gems, precompile assets and kick off your web server. Here's one I made earlier:

{% highlight text %}
FROM ruby:2.3
ENV RAILS_ENV=production
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install --deployment --without development --without test
COPY . ./
RUN bundle exec rake assets:precompile
EXPOSE 80
CMD ["bundle", "exec", "puma", "-b", "tcp://0.0.0.0", "-p", "80", "-e", "$RAILS_ENV"]
{% endhighlight %}

### Putting it all together

The final job is to put it all together in a docker-compose.yml file ready for deployment into your cluster. This will
have to include a few extra things from a standard compose file:

* Your app has to be constrained to worker nodes
* Your database has to be constrained to the storage node
* You have to specify a hostname which can be resolved the the frontend for your app containers (for interlock to work)

Here's my compose file as an example:

{% highlight yaml %}
version: "2"
services:
  rails-demo-db:
    image: "postgres:9.4"
    volumes:
      - pg_data:/var/lib/postgresql/data
    environment:
      - "constraint:node==storage"
      - POSTGRES_USER=example
      - POSTGRES_DATABASE=example
      - POSTGRES_PASSWORD=example

  rails:
    image: "your image address"
    hostname: "rails-demo.local"
    ports:
      - "80"
    environment:
      - "constraint:serverrole==apps"
      - DATABASE_NAME=example
      - DATABASE_PASSWORD=example
      - DATABASE_URL=postgres://rails-demo-db
      - DATABASE_USER=example
      - SECRET_KEY_BASE=aaaaaaaaaaaaaaaaaaaaa

volumes:
  pg_data:
    external: false
{% endhighlight %}

Now you can run `docker-compose up` against your cluster and it should bring up your application. Success. If you want
to view the app running then you'll need to add an entry to your hosts file (/private/etc/hosts on OSX) pointing to
your frontend node from `rails-demo.local`. Obviously in production you'll have real DNS names and pubic IPs so this
won't be necessary.

### Static assets

If you've deployed a Rails app before you'll probably be wondering why I'm letting Rails serve its own static files
instead of using a reverse proxy such as Nginx which does a far better job of it. Well there's one more piece to this
setup which I haven't mentioned yet, which is a caching server for your assets. Usually in production you'll use an
external paid service for this as they will do a good job of it with minimal effort. However if this is not an option
you can set it up yourself using Nginx.

The way this works is that you tell Rails to get your assets from the remote Nginx server. This server is configured to
work as a reveres proxy to your app, so it will send the request for an asset back to your app which will serve the
static asset to Nginx, which then sends it back to the client. The first time around this is a pretty bad way of doing
this, but combined with Nginx's caching config this becomes extremely efficient when anybody else asks for the asset.
using a few lines of config, you can get Nginx to keep hold of the asset it previously served ready for the next client
who comes asking. Here's my snipped of Nginx config which turns on this functionality.

{% highlight nginx %}
upstream rails_upstream {
    server rails-demo.local:80;
}

proxy_cache_path /tmp/nginx levels=1:2
                            keys_zone=my_cache:10m
                            max_size=10g
                            inactive=60m
                            use_temp_path=off;

server {
    server_name assets.rails-demo.local;

    location /assets {
        proxy_cache        my_cache;
        proxy_redirect     off;

        proxy_set_header   Host             rails-demo.local;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_pass         http://rails_upstream;
    }
}
{% endhighlight %}

The important thing with this config is that the `Host` header is set to the hostname of the upstream, otherwise
Interlock won't be able to figure out where to send the request. There's a whole lot of configuration you can use
with Nginx's caching, and what you see above is just taken from the example on their website.

Add this config into the [Nginx image](https://hub.docker.com/_/nginx/) and push it to your repo ready to deploy it.
You can add it to your compose file by adding the following block to your services:

{% highlight yaml %}
nginx:
  extra_hosts:
    - "rails-demo.local:<your frontend's IP>
  image: "your nginx image address"
  hostname: "assets.rails-demo.local"
  ports:
    - "80"
  environment:
    - "constraint:serverrole==apps"
{% endhighlight %}

Note that the `rais-demo.local` address had to be added manually because otherwise Nginx will have no idea where
to find the upstream. Again, you'll have to add the hostname `assets.rails-demo.local` to your hosts file to try this
out locally.

Now that nginx is ready we need to tell Rails to use our new static files cache to serve its files. Add this line to
`config/environments/production.rb` to do this:

{% highlight ruby %}
config.action_controller.asset_host = "http://assets.rails-demo.local"
{% endhighlight %}

And there you go. Updating your images using `docker-compose pull` followed by `docker-compose up` against the Swarm
should spin up your applicaiton, cache and database ready for use.

### Scaling

So the whole point of this was to get to a point where the application can be easily scaled horizontally. This is where
Docker Compose's `scale` function comes into play. It allows you to spin up (or down) any number of each service defined
in your `docker-compose.yml` file in an instant. So if your Rails app is starting to choke under the load, just run
`docker-compose scale rails=3` and 2 more will appear spread as evenly as possible across your worker nodes. Then when
load isn't as heavy and you want to free up some resources, `docker-compose scale rails=1` will cleanly tear down the
extra 2.

Interlock will handle the load balancing so you don't have to worry about that, and Nginx should be fine as a single
node since all it's doing is serving cached files (which the client's browser can also cache). Nginx can be scaled in
the same way as Rails was if need be though.

So there you go. A rails app scaled horiontally across multiple nodes with cached assets as well. There are a few tools
to get to grips with but nothing too complex, and once everything is up and running your app will be both as resilient
and as performant as you need.
