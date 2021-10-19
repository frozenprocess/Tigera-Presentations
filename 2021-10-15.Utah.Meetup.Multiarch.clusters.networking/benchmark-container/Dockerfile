FROM alpine:3.13
MAINTAINER Reza Ramezanpour <reza@projectcalico.org>

RUN apk update 
RUN apk add --no-cache wrk

WORKDIR '/app'

ENTRYPOINT ["wrk"]
