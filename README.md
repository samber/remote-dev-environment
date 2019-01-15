
# Development environment on remote server 🚀

This is a tuto for running containers on a remote server instead of local laptop.

- Control a remote docker daemon with DOCKER_HOST environment variable (and tls !)
- Port forwarding with SSH
- Volumes syncing with rsync (one way, from local to remote machine)

## Setup

### Remote machine

Launch a new server with docker.

New is better, because we will rsync a lot and things can break on remote machine if you rsync wrong directory ^^

Block every incoming connections to Docker containers in your firewall:

```sh
$ iptables -I DOCKER-USER -j DROP
$ iptables -I DOCKER-USER -i lo -j ACCEPT
$ iptables -I DOCKER-USER -i docker0 -j ACCEPT
$ iptables -I DOCKER-USER -m state --state ESTABLISHED,RELATED -j ACCEPT

# persist iptables config
$ iptables-save > /etc/iptables/rules.v4
$ reboot
```

### Local machine

```sh
$ brew install rsync lsyncd
```

```sh
# .bashrc

export REMOTE_DEV_SERVER=1.2.3.4

alias remote_docker='eval $(docker-machine env default) && echo OK'
alias remote_docker_reset='eval $(docker-machine env -u) && echo OK'

function remote_ports() {
  ports_list=$(pwd)/.ports
  size=$(cat ${ports_list} | wc -l)
  cat ${ports_list} | xargs -I {} echo Forwarding {}

  echo
  echo Enter ctrl-c to Quit
  echo

  cat ${ports_list} | xargs -P ${size} -I {} ssh -NL 127.0.0.1:{}:127.0.0.1:{} root@${REMOTE_DEV_SERVER}
}

function remote_sync()	{
  echo "Enter ctrl-c to quit\n"

  dir=$(pwd -P)
  tmpfile=$(mktemp /tmp/lsyncd.XXXXXX)

  cat <<EOF > ${tmpfile}
  settings {
    nodaemon   = true,
    insist     = true,
    maxProcesses = 1,
    maxDelays  = 0.5,
  }
  sync {
    default.rsyncssh,
    source="${dir}",
    host="${REMOTE_DEV_SERVER}",
    exclude=".git/",
    targetdir="${dir}",
    delete = true,
    rsync = {
      binary="/usr/local/bin/rsync",
      archive = true,
      compress = true,
      whole_file = false,
      owner = true,
      group = true,
      chown = "root:root",
    },
    ssh = {
      port = 22
    }
  }
EOF

  sudo lsyncd ${tmpfile}

  rm -f ${tmpfile}
}
```

```sh
# Setups docker-machine config, through SSH
$ docker-machine create --driver generic --generic-ip-address=${REMOTE_DEV_SERVER} default
Creating machine
...
```

## Running a project on a remote server

```sh
$ cd sample
```

```sh
# sync files between local machine and server
# first sync can be pretty long if you bring a node_modules/ directory ;)

$ remote_sync
14:27:52 Normal: --- Startup ---
14:27:52 Normal: recursive startup rsync: /a/b/c/ -> 1.2.3.4:/a/b/c/ excluding .git/
...
```

```sh
# link docker client to remote daemon
$ remote_docker
OK
$ docker-compose up
```

```sh
# Add ports separated by a new line into `.ports`.
echo "8080\n5432" > .ports

# forwards ports to localhost
$ remote_ports

Forwarding 8080
Forwarding 5432

Enter ctrl-c to Quit
```

## Security

Check iptables config:

```sh
# output

$ remote_docker
OK
# should work
$ docker run --rm -it debian ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=119 time=1.36 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=119 time=1.34 ms
```

```sh
# incoming ssh connections

# should work
ssh root@<ip>
```

```sh
# incoming docker connections

$ remote_docker
OK
$ docker run -d -p 8080:80 nginx
26201933b7ab94e7d333b9882d246d1b65b060e5dfe59d52d0cdefe5e8b17fcf
$ echo 8080 > .ports

$ remote_ports
Forwarding 8080

Enter ctrl-c to Quit

# should work
$ curl localhost:8080

# should fail
$ curl <server-ip>:8080
```
