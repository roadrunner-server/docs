FROM --platform=${TARGETPLATFORM:-linux/amd64} ghcr.io/roadrunner-server/roadrunner:latest as roadrunner
FROM --platform=${TARGETPLATFORM:-linux/amd64} php:8.3-alpine

COPY --from=roadrunner /usr/bin/rr /usr/local/bin/rr
COPY --from=mlocati/php-extension-installer:2 /usr/bin/install-php-extensions /usr/local/bin/

RUN install-php-extensions @composer-2 sockets protobuf

WORKDIR /src

COPY worker.php /src
COPY .rr.yaml /src
COPY composer.json /src

RUN composer install

ENTRYPOINT ["rr"]
