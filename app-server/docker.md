# Docker Images

Following Docker images are available:

| Description                              | Links                                                                             | Status                                                                                                                                                                                                                   |
|------------------------------------------|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Official RR image**                    | [Link](https://github.com/roadrunner-server/roadrunner/pkgs/container/roadrunner) | ![Latest Stable Version](https://img.shields.io/github/v/release/roadrunner-server/roadrunner.svg?maxAge=30) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) |
| **Third-party image from `n1215`**       | [Link](https://github.com/n1215/roadrunner-docker-skeleton)                       | [![License](https://poser.pugx.org/n1215/roadrunner-docker-skeleton/license)](https://packagist.org/packages/n1215/roadrunner-docker-skeleton)                                                                           |
| **Third-party image from `spacetab-io`** | [Link](https://github.com/spacetab-io/docker-roadrunner-php)                      | ![Latest Stable Version](https://img.shields.io/github/v/release/spacetab-io/docker-roadrunner-php) ![License](https://img.shields.io/github/license/spacetab-io/docker-roadrunner-php)                                  |

Here is an example of a `Dockerfile` that can be used to build a Docker image with RoadRunner for a PHP application:

{% code title="Dockerfile" %}

```dockerfile
FROM php:8.3-cli-alpine3.17 as backend

FROM spiralscout/roadrunner:2023.3 as roadrunner

# https://github.com/mlocati/docker-php-extension-installer
RUN --mount=type=bind,from=mlocati/php-extension-installer:2,source=/usr/bin/install-php-extensions,target=/usr/local/bin/install-php-extensions \
     install-php-extensions @composer-2 opcache zip intl sockets && \
     apk del --no-cache ${PHPIZE_DEPS} ${BUILD_DEPENDS}

EXPOSE 8080/tcp

WORKDIR /app

COPY --from=roadrunner /usr/bin/rr ./

ENV COMPOSER_ALLOW_SUPERUSER=1

# Copy composer files from app directory to install dependencies
COPY ./app/composer.* .

RUN composer install --optimize-autoloader --no-dev

# Copy application files
COPY ./app .

# Run RoadRunner server
CMD ./rr serve -c .rr.yaml
```

{% endcode %}
