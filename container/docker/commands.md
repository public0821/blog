To run an interactive shell in the Ubuntu image:

$ docker run -i -t ubuntu /bin/bash       
The -i flag starts an interactive container. The -t flag creates a pseudo-TTY that attaches stdin and stdout.

To detach the tty without exiting the shell, use the escape sequence Ctrl-p + Ctrl-q. The container will continue to exist in a stopped state once exited. To list all containers, stopped and running use the docker ps -a command.