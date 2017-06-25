To detach the tty without exiting the shell, use the escape sequence Ctrl-p + Ctrl-q. The container will continue to exist in a stopped state once exited. To list all containers, stopped and running use the docker ps -a command.


sudo ip link add veth0 type veth peer name veth1
sudo ip addr add 192.168.4.101/24 dev veth0
sudo ip addr add 192.168.4.102/24 dev veth1
sudo ip link set veth0 up
sudo ip link set veth1 up
ping -c 1 -I veth0 192.168.4.102