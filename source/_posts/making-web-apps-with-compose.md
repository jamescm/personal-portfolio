---
title: Making web apps with Docker Compose
date: 2018-06-29 18:09:45
tags: [tutorial, docker, docker compose, javascript, node, development, software]
---
I don't think it's any secret that I love working with Docker. Docker Compose is a layer that works really well for me as someone who often makes small web apps that don't necessitate the overhead provided by Kubernetes, Rancher, or any other container management system. Many of my Compose files contain fewer than 5 containers.

Today we'll be making a small Hello World app using Compose and Node. This is written from the perspective of developing in Ubuntu.

## File structure
```
.
├───web/
│   ├───index.js
│   └───Dockerfile
├───database.env
└───docker-compose.yml
```

## Making a simple app

Let's start by making a simple Express app in our `web` directory.

```javascript
// ./web/index.js
const express = require('express')
const app = express()
app.get('/', (req, res) => res.send('Hello, world!'))
app.listen(3000, () => console.log('Express listening on port 3000'))
```

We have to set up our `package.json` and dependencies too so let's do that now.

```sh
cd web
npm init -y
npm i -P express
npm shrinkwrap
rm -r node_modules
```

```json
{
  "name": "web",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "node .",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.16.3"
  }
}
```

Don't worry about the `npm shrinkwrap` statement yet, we'll get to that later. Now let's set up our `docker-compose.yml`.

```yaml
# ./docker-compose.yml
version: "3"
services:
  web:
    build: ./web
    ports:
      - "3000:3000"
    volumes:
      - ./web:/srv/app
    entrypoint: node .
```

In our Compose file, we've declared that we're using version 3 of Docker Compose and we have one service named `web`. We're going to build our container using a Dockerfile from `./web`. We've also mapped the host's port 3000 to the container's port 3000 so that we can access it using the host's IP address at port 3000 once we bring the container up. We're _voluming_ the contents of the host directory called `./web` into `/srv/app` on the container. Lastly, we've told Docker Compose to use `node .` as the entrypoint for the container. When the container starts, it will execute this command and give it PID 1. If PID 1 dies, the container shuts down.

That's a lot of stuff for just a few lines of code. Hopefully all these components will become a little easier to understand as we set up our app and make changes in the future.

So we've told Docker Compose to build from a Dockerfile in `./web` but we haven't written it yet! Let's do that now.

```Dockerfile
# ./web/Dockerfile
FROM node:10.5.0-alpine
COPY package.json npm-shrinkwrap.json /srv/app/
WORKDIR /srv/app
RUN npm i --production --quiet
```

Let's make a few changes to our Compose file again.

```yaml
# ./docker-compose.yml
...
  web:
    build: ./web
    ports:
      - "3000:3000"
    volumes:
      - ./web:/srv/app
      - /srv/app/node_modules
    working_dir: /srv/app
    entrypoint: node .
```

We've done a few things here to make our lives easier. This is also where our `npm shrinkwrap` becomes useful. We've built our Dockerfile to copy our `package.json` and `npm-shrinkwrap.json` into the app location in our container and then install our dependencies into the image. Now we won't have to resort to voluming in our `node_modules` from the host to the container. For a better understanding of what's happening here, I _highly_ recommend reading [this article by John Lees-Miller](https://jdlm.info/articles/2016/03/06/lessons-building-node-app-docker.html).

If you run `docker-compose up` and everything went right, you should see the following output. You can also now visit your site to see "Hello, world!"

```
web_1       | > web@1.0.0 start /srv/app
web_1       | > node .
web_1       |
web_1       | Express listening on port 3000
```

## Using a database

Now that'd be great if web apps could be made with a few lines in Express but unfortunately that's not the world we live in. Let's give our app a database to work with. Before we go modify our Compose file, let's dive into the [`.env` file](https://docs.docker.com/compose/env-file). To put it *very* simply, it's a file that Docker automatically reads in order to load environment variables. Let's use ours to help with development. It is also possible to specify custom env files in Docker. In Compose, we do it with [`env_file`](https://docs.docker.com/compose/environment-variables/#the-env_file-configuration-options). We're going to use a custom env file instead of a `.env` since we're going to pass the same variables to our database and our web app so this'll keep it [DRY](http://wiki.c2.com/?DontRepeatYourself).

```sh
# ./database.env
MYSQL_USER=myUser
MYSQL_PASSWORD=myPassword
MYSQL_DATABASE=myDatabase
```

Now let's add our database to compose.

```yaml
# ./docker-compose.yml
version: "3"
services:
  database:
    image: mysql:5.7.22
    volumes:
      - mysql_data:/var/lib/mysql
    environment:
      - MYSQL_RANDOM_ROOT_PASSWORD=yes
    env_file:
      - database.env

  web:
    build: ./web
    env_file:
      - database.env
    ports:
      - "3000:3000"
    volumes:
      - ./web:/srv/app
      - /srv/app/node_modules
    entrypoint: node .

volumes:
  mysql_data:
```

We've added a new container using `mysql:5.7.22` and given it a few environment variables from our `database.env` file. Under volumes, we mount the named volume `mysql_data` to `/var/lib/mysql` which is where the container stores its data. You'll also notice at the bottom is where we declare our named volume. All this is so we can persist our database in the case the container is ever destroyed.

Let's fire it up with `docker-compose up`. You're going to see a lot of gibberish from MySQL but it should end with something like the following.

```
database_1  | 2018-06-29T22:06:42.590438Z 0 [Note] mysqld: ready for connections.
database_1  | Version: '5.7.22'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
```

Let's try connecting to our database from our app using Sequelize. First, install Sequelize and mysql2 as a production dependency.

```sh
sudo rm -r ./web/node_modules
npm --prefix ./web i -P sequelize mysql2
docker-compose build web
docker-compose rm -fs web
```

Next we'll try connecting to it.

```javascript
// ./web/index.js
const express = require('express')
const Sequelize = require('sequelize')
const { MYSQL_USER, MYSQL_PASSWORD, MYSQL_DATABASE } = process.env
const sequelize = new Sequelize(MYSQL_DATABASE, MYSQL_USER, MYSQL_PASSWORD, {
  host: 'database', // We're using the container name as a host name
  dialect: 'mysql'
})

const app = express()
app.get('/', async (req, res) => {
  try {
    await sequelize.authenticate()
    res.send('Hello from MySQL!')
  } catch (err) {
    res.send(err)
  }
})
app.listen(3000, () => console.log('Express listening on port 3000'))
```

Run everything with `docker-compose up` and visit your site which is presumably at `http://localhost:3000`. If all went well you should see "Hello from MySQL!" in your browser and the following in your web logs.

```
web_1       | Executing (default): SELECT 1+1 AS result
```

This bit of magic was enabled by [Compose networking](https://docs.docker.com/compose/networking/). In short, Docker Compose links all of your containers into a single network that can communicate with one another by using the container name given to them in the Compose file. So when we pass 'database' to the host argument in Sequelize, that eventually gets resolved to the Docker IP for the database container. Neat, right?

I won't pretend like this is at all best practice for developing a production app in Docker. We could further secure our MySQL passwords by storing them in the machine's environment variables rather than an env file. We could also declare our app to be a Docker swarm and use `docker secrets`. In addition, our application is running as root from within our container; it'd be better if it ran as a user with less privileges. We could also improve our `node_modules` situation by assigning it a named volume as well. We will investigate some improvements in [part 2](/personal-portfolio/2018/06/30/making-web-apps-with-compose-2)!

## Read more

Part 1: Docker Compose and Node: Getting started
Part 2: [Development to Production: Security](/personal-portfolio/2018/07/05/making-web-apps-with-compose-2)
Part 3: Development to Production: Availability

Check this example out in Git: https://github.com/jamescm/docker-web-app-example.
