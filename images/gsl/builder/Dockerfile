FROM alpine:3.5

RUN apk add --no-cache \
                       gcc \
                       make \
                       musl-dev \
                       pcre-dev

WORKDIR /src

ENTRYPOINT ["make", "-C", "src"]
