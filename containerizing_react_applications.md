[//]: (I'm not sure why, but I've actually setup quiete a lot react js stacks just recently. Because of this, I got some experience in the "lowest common denominator" of nearly every react stack when it comes to container deployments. This article mainly focuses in deploying client side react applications - I have tried different ways of server side rendering in a node js container, but I always experienced not acceptable memory consumptions in such applications. So I decided to focus on regular, non universal react apps for now and share some small snippets.)
I'm not sure why, but I've actually setup quiete a lot react js stacks just recently. Because of this, I got some experience in the "lowest common denominator" of nearly every react stack when it comes to container deployments. This article mainly focuses in deploying client side react applications - I have tried different ways of server side rendering in a node js container, but I always experienced not acceptable memory consumptions in such applications. So I decided to focus on regular, non universal react apps for now and share some small snippets.

## Prerequisites
This article doesn't really require knowledge of docker or a specific webserver, although its quiete useful to understand what the scripts are doing and what the config means. But you should have docker installed of course.
If the stack you want to bake into a container contains the possibility to install all requirements via `npm install` and create a dist folder with all the static assets via e.g. `npm build` or a similar command, you'll get a full build pipeline of your application. But at the end, the container needs to be pushed into a central docker registry - the easiest way to do that is by creating an account on [dockerhub]. As I just noticed at the time of writing, you even get 1 private repo for free.

## Project structure
There is no good react stack out there, without a complex develop and build pipeline. Since they're mostly powered by tools like [grunt], [gulp] or [webpack], most projects come with a handy way to trigger this build pipeline with [npm scripts]. This pipelines mostly involve some [babel] transformation (just because ES6 is cool ;-) ), some minify and uglify routines and a way to create a "production bundle". If your stack can handle these tasks and ideally create a _dist/_ folder with all the required files for the website, setting up a production grade container is a matter of minutes.
This is the project structure I will use for this post, although there are typically many more files in the root folder (like .eslint or .babelrc), and many more folders depending on your setup, there is no difference for this and I decided to go with the smallest example:

```
src/
_dist/_
build/
packages.json
```

The only important part for the container is the `dist` folder. This folder will live in the document root of the webserver in your container. As long as you are capable of building such a folder, you don't have any other requirement for the container. 

Let's add a few files to your project:

```
src/
_dist/_
build/
*Caddyfile*
*Makefile*
*Dockerfile*
packages.json
```

### Caddyfile
This file is the project specific configuration of your webserver. In our case, we use [Caddy] - I'll come back to the question _why?_ in a bit.
```
:2015
gzip

root /var/www/

rewrite {
    if {file} not favicon.ico
    if {file} not *.css
    if {file} not *.js
    if {file} not *.png
    if {file} not *.jpg
    if {file} not *.jpeg
    to {path} {path}/ /index.html
}
```

I think this config is pretty self-explaining, even if you've never heard from [Caddy] before. One of the reasons why I love it :-)
The rewrites are necessary if someone hits F5 after navigating to another route. Per default, [Caddy] will try to find the file of your route on the filesystem (as every other webserver btw), which will not work for many ressons in a react app. The document root and the port is what need to be reflected in the `Dockerfile` too, just in case you want to change it. If you have svgs or something similar in your project, just add the suffix to the rewrite block.

### Dockerfile
I personally like to always use the [alpine] base image if possible - so I do for the Docker container of this app.

```
FROM alpine

WORKDIR /usr/src

RUN apk update && apk add ca-certificates && apk add wget && update-ca-certificates 
RUN	wget https://github.com/mholt/caddy/releases/download/v0.8.2/caddy_linux_amd64.tar.gz -O /usr/src/caddy.tar.gz && \
	tar xvzf /usr/src/caddy.tar.gz && \
	rm -rf /usr/src/caddy.tar.gz

ADD Caddyfile /usr/src/Caddyfile
ADD dist/ /var/www/

EXPOSE 2015

CMD ["/usr/src/caddy"]
``` 

It basically does nothing more than installing some certificates and [Caddy]. The `Caddyfile` will be copied into the container to configure the webserver and the generated _dist/_ directory to serve the application from the document root. Than we just expose the port from the `Caddyfile` and run the process.

### Makefile
You can find `Makefile`s in pretty much every project of mine - I always use it because I'm lazy (you need to see my aliases ;-) ).
From shellscripts to an ant.xml could everything do this job. But that's also the reason why you can use it without knowing about docker.

```
USER=foo
NAME=bar
VERSION=0.0.1
REGISTRY_URL=$(USER)/$(NAME)

pwd=$(shell pwd)

docker:
	docker build --no-cache=true -t $(REGISTRY_URL):latest .

run:
	docker run -d --name frontend -p 2015:2015 $(REGISTRY_URL):latest

stop:
	docker stop frontend && docker rm frontend

sh:
	docker exec -it frontend /bin/sh

push:
	@echo $(VERSION)
	@echo $(REGISTRY_URL)
	@docker tag -f "$(REGISTRY_URL):latest" "$(REGISTRY_URL):$(VERSION)"
	@docker tag -f "$(REGISTRY_URL):latest" "$(REGISTRY_URL):latest"
	@docker push "$(REGISTRY_URL):$(VERSION)"
	@docker push "$(REGISTRY_URL):latest"

clean:
	rm -rf ./dist && rm -rf ./node_modules
```

Just modify the upper variables to fit your needs. If you use [dockerhub], you don't need a URL for the registry. If you use a private registry or another public service for this, just modify the `REGISTRY_URL`.
The `push` task creates 2 tags of the container, one `latest` tag and a tag specified in `VERSION`. This is common practise, to bump latest always to the latest version of the image. So every push points to the current version.

### The full pipeline
Now we have everything to build and run the container, so how would a Jenkins job look like?

```
#!/bin/bash
cd $WORKSPACE
npm build
make docker
make push
```

The `Makefile` includes some basic tasks you might need for local testing. `make run` runs the container, `make sh` opens a console and `make stop` stops and deletes the container.

## Why Caddy?
There's a reason, why I build everything on top of [alpine] - just because the smaller the container you run, the faster is your startup. Locally, and even more important, in a kubernetes (or similar orchestration system) environment. 
The total size of this blog (with kind of exactly this setup) is about *25MB*. That is super small and super fast. The Webserver does not much, it simply need to serve some static files. That's a job every webserver can handle easily. But if you want to use some handy features from [Caddy] such as the [automatic HTTPS], just modify the `Caddyfile` according to your needs.

[alpine]: https://hub.docker.com/_/alpine/
[automatic HTTPS]: https://caddyserver.com/docs/automatic-https
[Caddy]: https://caddyserver.com
[webpack]: https://webpack.github.io/
[grunt]: http://gruntjs.com/
[gulp]: http://gulpjs.com/
[dockerhub]: https://hub.docker.com
[npm scripts]: https://docs.npmjs.com/misc/scripts
[babel]: https://babeljs.io/