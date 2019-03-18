# Tips and tricks around docker

# Cleanup docker containers, images, ...
Sometimes it is necessary to cleanup docker containers, images
or anything related, which is not needed/not running anymore.
Docker comes with a system-wide clean-up command:

    docker system prune

This cleanes up non-running containers, unused network connections,
images and build caches. However, it does not clean volumes without
explicitely adding the --volumes command line flag.

To cleanup everything, all containers need to be stopped.
The following command gets a list of all contaienrs and displays only the IDs:

    docker container ls -aq

To do a complete clean-up. stop all containers and do a prune:

    docker container stop $(docker container ls -a -q) && docker system prune -a -f --volumes

