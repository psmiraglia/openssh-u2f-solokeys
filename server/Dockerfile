FROM debian:bullseye

LABEL maintainer="paolo.smiraglia@gmail.com"

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        openssh-server \
    && mkdir /var/run/sshd

COPY ./rootfs/ /

EXPOSE 2222/tcp
CMD ["/usr/sbin/sshd", "-Dd"]
