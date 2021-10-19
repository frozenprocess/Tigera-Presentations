FROM alpine:3.13
MAINTAINER Reza Ramezanpour <reza@projectcalico.org>

RUN apk update 
RUN apk add --no-cache nginx

WORKDIR '/var/www/localhost/htdocs'

RUN touch 0kb.bin
RUN dd if=/dev/zero of=1kb.bin bs=1KB count=1
RUN dd if=/dev/zero of=1mb.bin bs=1MB count=1

COPY default.conf /etc/nginx/http.d/default.conf 
COPY index.html .
RUN mkdir /run/nginx
CMD ["-g", "daemon off;"]
ENTRYPOINT ["nginx"]
