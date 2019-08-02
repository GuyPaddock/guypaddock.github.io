---
layout: article
title: "Docker Build-time Environment Variable Precedence in Multi-stage Builds"
categories: posts
modified: 2019-08-01T11:15:00-04:00
tags: [docker, gotchas, environment-variables, multi-stage-builds, drupal, drush]
comments: true
ads: false
---
I just learned the hard way that the following won't work:
- You create (or derive from) a Docker base image that defines a default value for an environment variable.
- You have a multi-stage build.
- You need to override the environment variable of the base image during the build of the child image.

Instead, the only workable solution it seems is to remove the environment variable from the base image in
order to set it at build time in your child image.

To illustrate, here's the `Dockerfile` for the base image (`inveniem/drush`):

```Dockerfile
FROM drush/drush:8

COPY entrypoint-wrapper.sh /bin/entrypoint-wrapper.sh
RUN chmod +x /bin/entrypoint-wrapper.sh

ARG SSH_KEYS
ENV SSH_KEYS="${SSH_KEYS}"

RUN mkdir -p /root/.ssh && \
  chmod 700 /root/.ssh && \
  ssh-keyscan -H gitlab.com >> ~/.ssh/known_hosts && \
  ssh-keyscan -H 35.231.145.151 >> ~/.ssh/known_hosts

ENTRYPOINT ["/bin/entrypoint-wrapper.sh"]
```

`entrypoint-wrapper.sh` allows us to pass-in SSH keys during container build-time to access private repos. With an echo-back I added to debug the issue with environment variables, the file looks like this:
```bash
#!/usr/bin/env bash

set -e

# Echo environment variables for DEBUGGING purposes
# REMOVE THIS to avoid displaying SSH keys
export

if [[ ! -z "${SSH_KEYS:-}" ]]; then
  echo "${SSH_KEYS}" > /root/.ssh/id_rsa

  chmod 600 /root/.ssh/id_rsa
fi

drush "$@"
```

And here's a multi-stage `Dockerfile` for the child image:

```Dockerfile
################################################################################
# Build Theme
################################################################################
FROM node:7.5.0 as theme_compile

# ... omitted the theme build instructions ...

################################################################################
# Build Installation Profile
################################################################################
FROM inveniem/drush:latest as profile_compile

COPY --from="theme_compile" /theme /app/themes/custom/our_theme
COPY build/app /app

# Gitlab SSH key (without passphrase)
ARG SSH_KEYS
ENV SSH_KEYS="${SSH_KEYS}"

RUN /bin/entrypoint-wrapper.sh make build-platform.make /app-build

################################################################################
# Build Drupal Runtime
################################################################################
FROM drupal:7-apache

RUN rm -rf /var/www/html

COPY --from="profile_compile" --chown=www-data:www-data \
  /app-build /var/www/html

VOLUME /var/www/html/sites
```

Seems pretty straightforward, but doesn't work. When the child `Dockerfile` is invoked with `docker build --build-arg 'SSH_KEYS=<<SSH KEY GOES HERE>>' .`, the `SSH_KEYS` build-time argument is supposed to be passed into the `SSH_KEYS` environment variable needed by our custom Drush container image. That image, in-turn, is supposed to use [Drush Make](https://docs.drush.org/en/8.x/make/) to assemble an installation profile for us in one container which we then copy into the final container image that's based on the Drupal 7 Apache container image. This helps to protect the credentials (SSH keys) that are used during the build from leaking into the final image.

Instead, `SSH_KEYS` is always empty inside the Drush container. This happens _even if_ the `RUN` instruction in the child `Dockerfile` is changed to this:

```Dockerfile
RUN SSH_KEYS="${SSH_KEYS}" /bin/entrypoint-wrapper.sh make build-platform.make /app-build
```

This is because, as soon as the `FROM inveniem/drush:latest` instruction is evaluated, the environment gets replaced with the environment from that base image. This feels like a bug, but I think it makes sense since the base image's definition of the
variable takes precedence over what was in the environment before-hand.

The fix is paradoxically to remove the environment variable definitions from the base image, as follows:

```Dockerfile
FROM drush/drush:8

COPY entrypoint-wrapper.sh /bin/entrypoint-wrapper.sh
RUN chmod +x /bin/entrypoint-wrapper.sh

# Removed these next two lines to allow SSH_KEYS to be set during build of child image:
# ARG SSH_KEYS
# ENV SSH_KEYS="${SSH_KEYS}"

RUN mkdir -p /root/.ssh && \
  chmod 700 /root/.ssh && \
  ssh-keyscan -H gitlab.com >> ~/.ssh/known_hosts && \
  ssh-keyscan -H 35.231.145.151 >> ~/.ssh/known_hosts

ENTRYPOINT ["/bin/entrypoint-wrapper.sh"]
```
