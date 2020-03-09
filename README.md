# Implement FIDO U2F in OpenSSH with SoloKeys

According to [release notes](https://www.openssh.com/txt/release-8.2),
OpenSSH 8.2 introduces the support for
[FIDO Universal 2nd Factor (U2F)](https://fidoalliance.org/specifications).
[SoloKeys](https://solokeys.com) are FIDO2 security keys so, why not using
them to test the new OpenSSH feature?

**Note:** The tutorial considers as starting point a minimal fresh
installation of Ubuntu 18.04 and a Solo key with firmware at 3.1.2 version.

## Dependencies

Install dependencies to build OpenSSH from sources

    $ sudo apt-get install build-essential libcbor-dev libssl-dev zlib1g-dev

Install [`libfido2`](https://github.com/Yubico/libfido2)

    $ sudo apt-add-repository ppa:yubico/stable
    $ sudo apt update
    $ sudo apt-get install libfido2-dev libfido2-udev

## Build OpenSSH

Get the sources

    $ wget https://openbsd.mirror.garr.it/pub/OpenBSD/OpenSSH/portable/openssh-8.2p1.tar.gz
    $ tar zxvf openssh-8.2p1.tar.gz
    $ cd openssh-8.2p1

Build and install OpenSSH under `/home/user/openssh`

    $ ./configure --prefix=/home/user/openssh --with-security-key-builtin
    $ make
    $ make install

Verify the installation

    $ ~/openssh/bin/ssh -V
    OpenSSH_8.2p1, OpenSSL 1.1.1  11 Sep 2018

## Generate a keypair

Generate an ECDSA keypair

    $ ./openssh/bin/ssh-keygen -vvvv -t ecdsa-sk -C "My Solo Key"
    Generating public/private ecdsa-sk key pair.
    You may need to touch your authenticator to authorize key generation.
    debug3: start_helper: started pid=12093
    debug3: ssh_msg_send: type 5
    debug1: start_helper: starting /home/user/openssh/libexec/ssh-sk-helper
    debug3: ssh_msg_recv entering
    debug1: sshsk_enroll: provider "internal", device "(null)", application "ssh:", userid "(null)", flags 0x01, challenge len 0
    debug1: sshsk_enroll: using random challenge
    debug1: ssh_sk_enroll: using device /dev/hidraw1

    (...press your Solo key...)

    debug3: ssh_sk_enroll: attestation cert len=775
    debug1: ssh-sk-helper: reply len 1102
    debug3: ssh_msg_send: type 5
    debug3: reap_helper: pid=12093
    Enter file in which to save the key (/home/user/.ssh/id_ecdsa_sk):
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    Your identification has been saved in /home/user/.ssh/id_ecdsa_sk
    Your public key has been saved in /home/user/.ssh/id_ecdsa_sk.pub
    The key fingerprint is:
    SHA256:diHAp8OzNzFDfTr+kLu4RTpSlYggDx7SjF1FdJsJWFY SoloKeys
    The key's randomart image is:
    +-[ECDSA-SK 256]--+
    |.=+.oBO.E.       |
    |.o+=.o.=o=...    |
    |  . ...+* +o     |
    |      = +oo.     |
    |       +S=oo     |
    |      .oo++      |
    |      ..o..+     |
    |       . +. .    |
    |        o...     |
    +----[SHA256]-----+

Append the pubkey to the `authorized_keys` file

    $ cat .ssh/id_ecdsa_sk.pub >> .ssh/authorized_keys

## Run the demo

Run the `sshd` daemon in foreground (`-D`) with debug mode enabled (`-d`) and
bound to an alternative port (`-p 2222`)

    $ /home/user/openssh/sbin/sshd -d -D -p 2222
    debug1: sshd version OpenSSH_8.2, OpenSSL 1.1.1  11 Sep 2018
    debug1: private host key #0: ssh-rsa SHA256:/Suom4amAqBzq7sN0qbDgsum6/owzbfyvavlt1Y116s
    debug1: private host key #1: ecdsa-sha2-nistp256 SHA256:7bYdBHpS3hz+GT5VXakyE++zaKiTzVnbywKkA6SXpCE
    debug1: private host key #2: ssh-ed25519 SHA256:EtA3mECbpO4aRTFr0Af08MVGw7cu9AG8Q6pWDuBN670
    debug1: setgroups() failed: Operation not permitted
    debug1: rexec_argv[0]='/home/user/openssh/sbin/sshd'
    debug1: rexec_argv[1]='-dD'
    debug1: rexec_argv[2]='-p'
    debug1: rexec_argv[3]='2222'
    debug1: Set /proc/self/oom_score_adj from 0 to -1000
    debug1: Bind to port 2222 on 0.0.0.0.
    Server listening on 0.0.0.0 port 2222.
    debug1: Bind to port 2222 on ::.
    Server listening on :: port 2222.

Alternatively, you can build and run a Docker image that implements an
OpenSSH server

    $ cd server
    $ cat .ssh/id_ecdsa_sk.pub >> rootfs/root/.ssh/authorized_keys
    $ sudo docker build --tag myssh .
    $ sudo docker run -t --rm --name myssh -p 2222:2222 myssh
    debug1: sshd version OpenSSH_8.2, OpenSSL 1.1.1d  10 Sep 2019
    debug1: private host key #0: ssh-rsa SHA256:z/oEq7K935t9dJ2uDMAELeXFloO0ubYp2zA+oLbLLac
    debug1: private host key #1: ecdsa-sha2-nistp256 SHA256:1X/seu5/F4YYXlwz4d/arqBisR0iE9jXFKzeypzgqgM
    debug1: private host key #2: ssh-ed25519 SHA256:FT9DIarCeG5p9+1FYSsxXLwmI3wN3jOK+ImjU8oT9PE
    debug1: rexec_argv[0]='/usr/sbin/sshd'
    debug1: rexec_argv[1]='-Dd'
    debug1: Set /proc/self/oom_score_adj from 0 to -1000
    debug1: Bind to port 2222 on 0.0.0.0.
    Server listening on 0.0.0.0 port 2222.
    debug1: Bind to port 2222 on ::.
    Server listening on :: port 2222.

Open another terminal and try to login

    $ ./openssh/bin/ssh -l user -p 2222 -i .ssh/id_ecdsa_sk localhost
    Confirm user presence for key ECDSA-SK SHA256:diHAp8OzNzFDfTr+kLu4RTpSlYggDx7SjF1FdJsJWFY

    (...press your Solo key...)

    Last login: Mon Mar  2 17:30:37 2020 from ::1
    Environment:
      USER=user
      LOGNAME=user
      HOME=/home/user
      PATH=/usr/bin:/bin:/usr/sbin:/sbin:/home/user/openssh/bin
      MAIL=/var/mail/user
      SHELL=/bin/bash
      TERM=xterm-256color
      SSH_CLIENT=::1 47732 2222
      SSH_CONNECTION=::1 47732 ::1 2222
      SSH_TTY=/dev/pts/3

Enjoy with SoloKeys, OpenSSH and FIDO2 U2F!!!
