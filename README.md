# Shepherd

[![Build Status](https://ci.strahlungsfrei.de/api/badges/djmaze/shepherd/status.svg)](https://ci.strahlungsfrei.de/djmaze/shepherd)
[![Docker Stars](https://img.shields.io/docker/stars/mazzolino/shepherd.svg)](https://hub.docker.com/r/mazzolino/shepherd/) [![Docker Pulls](https://img.shields.io/docker/pulls/mazzolino/shepherd.svg)](https://hub.docker.com/r/mazzolino/shepherd/)

This fork builds on the brilliant original by added a wget at the end of the script when a service is updated. This wget is customisable and is intended to call a webhook of your choosing (I am using one from Zapier Webhooks) and enable the event to be published wherever you like - for example on a channel on Slack or Rocket.Chat.

## Usage

    docker service create --name shepherd \
                          --constraint "node.role==manager" \
                          --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock,ro \
                          mazzolino/shepherd

## Or with docker-compose
    version: "3"
    services:
      ...
      shepherd:
        build: .
        image: mazzolino/shepherd
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        deploy:
          placement:
            constraints:
            - node.role == manager

### Configuration

Shepherd will try to update your services every 5 minutes by default. You can adjust this value using the `SLEEP_TIME` variable.

You can prevent services from being updated by appending them to the `BLACKLIST_SERVICES` variable. This should be a space-separated list of service names.

Alternatively you can specify a filter for the services you want updated using the `FILTER_SERVICES` variable. This can be anything accepted by the filtering flag in `docker service ls`.

You can enable private registry authentication by setting the `WITH_REGISTRY_AUTH` variable.

You can receive a post-update event by providing a `POST_UPDATE_URL`.

Example:

    docker service create --name shepherd \
                        --constraint "node.role==manager" \
                        --env SLEEP_TIME="5m" \
                        --end POST_UPDATE_URL=""\
                        --env BLACKLIST_SERVICES="shepherd my-other-service" \
                        --env WITH_REGISTRY_AUTH="true" \
                        --env FILTER_SERVICES="label=com.mydomain.autodeploy"
                        --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock,ro \
                        --mount type=bind,source=/root/.docker/config.json,target=/root/.docker/config.json,ro \
                        mazzolino/shepherd

## How does it work?

Shepherd just triggers updates by updating the image specification for each service, removing the current digest.

Most of the work is thankfully done by Docker which [resolves the image tag, checks the registry for a newer version and updates running container tasks as needed](https://docs.docker.com/engine/swarm/services/#update-a-services-image-after-creation).

Also, Docker handles all the work of [applying rolling updates](https://docs.docker.com/engine/swarm/swarm-tutorial/rolling-update/). So at least with replicated services, there should be no noticeable downtime.
