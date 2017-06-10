Borg Server Container
=====================

Description
-----------

Basic Borg backup server container. It uses `sebcworks/borgbackup-base` as base image and will launch sshd in foreground.

It is supposed to have a ssh `authorized_keys` file mounted at `/home/borg/.ssh/`, and will use `/var/backups/borg` as its repository folder.

Usage
-----

You are free to put the data where you want, but I recommand this setup (with Docker 1.9+):

1. Create a volume to hold `authorized_keys` file
2. Mount this volume in a temporary `sebcworks/borgbackup-base` container to create the `authorized_keys` file
3. Mount a host folder or a volume in a temporary `sebcworks/borgbackup-base` container to initialize the repository
4. Mount the volume in the `sebcworks/borgbackup-server` container, as well as the folder/volume to hold the repository

**WARNING** With Docker 1.9, mounting a volume created through the `docker volume create` command will be deleted if mounted to a container
launched with the --rm flag set (`docker run --rm -v named-volume:/data ...` will delete named-volume at exit). This behavior has changed
in the 1.10 version.

**WARNING 2** If you use a host folder mapped into the container as below, be sure that this folder is under a folder owned by *root:root* with *0750* rights to avoid
security problem (as the host user with the uid equivalent to the uid of borg inside the container will own the host folder).

    docker volume create --name borgbackup-data-authorizedkeys
    docker run -it --name borg-init-ssh-server -v borgbackup-data-authorizedkeys:/sshdata sebcworks/borgbackup-base /bin/bash
    > apt update && apt install nano
    > nano /sshdata/authorized_keys
    -> Paste the public key of the "borg" user from the client
    -> Put **BEFORE** the previous content: command="cd /var/backups/borg; borg serve --restrict-to-path /var/backups/borg",no-pty,no-agent-forwarding,no-port-forwarding,no-X11-forwarding,no-user-rc
    > chown borg.borg /sshdata/authorized_keys
    > exit
    docker rm borg-init-ssh-server

    [OPTIONAL: Initialize a repository]
    docker run -it --name borg-init-repos-server -v /path/to/my/repos:/var/backups/borg sebcworks/borgbackup-base /bin/bash
    > su borg
    > borg init --encryption=repokey /var/backups/borg
    > exit
    docker rm borg-init-repos-server
    [/OPTIONAL]

    docker run --name borg-server -v borgbackup-data-authorizedkeys:/home/borg/.ssh -v /path/to/my/repos:/var/backups/borg -p 2222:22 -d sebcworks/borgbackup-server