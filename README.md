# Docker SNX Checkpoint VPN

## Introduction

Client for Checkpoint VPN using snx GNU/Linux client. It accepts username and/or certificate.

## Usage

### 1. Create the container

* With `docker-compose` (`.p12` certificate file required)

    1. Create a `.env` file

        Create a `.env` file in the root of the project, alongside [`docker-compose.yml`](docker-compose.yml), with your env variables (take a look at  [`.env.example`](.env.example)).

    2. Start the container

        ```sh
        docker-compose up -d
        ```

OR

* With `docker`

    **IMPORTANT**: to use your certificate, specify a volume with `/certificate.p12` as container path

    1. With certificate

        ```sh
        docker run --name snx-vpn \
        --cap-add=ALL \
        -v /lib/modules:/lib/modules \
        -e SNX_SERVER=vpn_server_ip_address \
        -e SNX_PASSWORD=secret \
        -v /path/to/my_snx_vpn_certificate.p12:/certificate.p12 \
        -t \
        -d ananni/snx-checkpoint-vpn
        ```

    2. With certificate and username

        ```sh
        docker run --name snx-vpn \
        --cap-add=ALL \
        -v /lib/modules:/lib/modules \
        -e SNX_SERVER=vpn_server_ip_address \
        -e SNX_USER=user \
        -e SNX_PASSWORD=secret \
        -v /path/to/my_snx_vpn_certificate.p12:/certificate.p12 \
        -t \
        -d ananni/snx-checkpoint-vpn
        ```

    3. Without certificate (NOT TESTED)

        ```sh
        docker run --name snx-vpn \
        --cap-add=ALL \
        -v /lib/modules:/lib/modules \
        -e SNX_SERVER=vpn_server_ip_address \
        -e SNX_USER=user \
        -e SNX_PASSWORD=secret \
        -t \
        -d ananni/snx-checkpoint-vpn
        ```

### 2. Get the container IP address

```sh
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' snx-vpn
```

Example output:

```sh
172.17.0.2
```

### 3. Add a route using the container IP address as gateway

```sh
sudo route add -net <ip> gw 172.17.0.2 netmask 255.255.255.0
```

Replace `<ip>` with the IP address you want to reach with the VPN.

If netmask `255.255.255.0` isn't working, you can try `255.255.255.255`.

### 4. Try to reach the server behind SNX VPN (e.g. through SSH)

```sh
ssh <ip>
```

## Environment Variables

`SNX_SERVER`: Mandatory. IP address or name of the Checkpoint VPN server

`SNX_PASSWORD`: Mandatory. String corresponding to the password of VPN client

`SNX_USER`: Optional if certificate volume has been provided, otherwise mandatory. String corresponding to the username of VPN client

With `docker-compose` you also need the following variables:

`CONTAINER_NAME`: Name to assign to the container

`LOCAL_CERTIFICATE_PATH`: Local absolute path to your `.p12` certificate

**IMPORTANT**: Remember to escape any special character. For example, `stR0ngPas$word2` becomes `'stR0ngPas\$word2'`. Wrapping with single quotes is only needed if you're creating the container with `docker run`, you can omit them if you're using `docker-compose` with the `.env` file.

## Allowed volumes

`/certificate.p12`: A VPN client certificate. If present the SNX binary will be invoked with `-c` parameter pointing to this certificate file.

## Routes

Since the container is the one that connects to the VPN server, it's the one that receives the routes. In order to list all of them perform the following command from the docker host (`snx-vpn` is the container name in this example):

```sh
docker exec -ti snx-vpn route -n | grep -v eth0
```

Expected output similar to:

```sh
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.20.30.0       0.0.0.0         255.255.255.0   U     0      0        0 tunsnx
10.20.40.0       0.0.0.0         255.255.255.0   U     0      0        0 tunsnx
...
```

Then manually add a route:

```sh
sudo route add -net 10.20.30.0 netmask 255.255.255.0 gw `docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' snx-vpn`
```

And finally test access. In this example trying to reach via SSH a remote server:

```sh
ssh user@10.20.30.40
```

## DNS

Since the container is the one that connects to the VPN server, it's the one that receives the DNS servers. In order to get them you can proceed in on of two ways:

* Check the container logs (`snx-vpn` is the container name in this example)

    ```sh
    docker logs snx-vpn | grep DNS
    ```

    Expected output similar to:

    ```sh
    DNS Server          : 10.20.30.11
    Secondary DNS Server: 10.20.30.12
    ```

OR

* Check `/etc/resolv.conf` container file (`snx-vpn` is the container name in this example)

    ```sh
    docker exec -ti snx-vpn cat /etc/resolv.conf
    ```

    Expected output similar to:

    ```sh
    nameserver 10.20.30.11
    nameserver 10.20.30.12
    nameserver 8.8.4.4
    nameserver 8.8.8.8
    ```

Once you know the DNS servers you can proceed in one of two ways:

* Update your docker host `/etc/resolv.conf`

    ```sh
    sudo vim /etc/resolv.conf
    ```

    With below content:

    ```sh
    nameserver 10.20.30.11
    nameserver 10.20.30.12
    ```

    You should remember to revert back the changes once finished

OR

* Run a local dnsmasq service. It requires that you know the remote domains beforehand (`example.com` in this example)

    1. Create the file:

        ```sh
        sudo vim /etc/dnsmasq.d/example.com
        ```

        With below content:

        ```sh
        server=/example.com/10.20.30.11
        ```

    2. Restart the "dnsmasq" service

        ```sh
        sudo service dnsmasq restart
        ```

    3. Test it

        ```sh
        ssh server.example.com
        ```

## Troubleshooting

This image has just been tested without username and with certificate, and with snx build 800010003 obtained from:

<https://supportcenter.checkpoint.com/supportcenter/portal/user/anon/page/default.psml/media-type/html?action=portlets.DCFileAction&eventSubmit_doGetdcdetails=&fileid=22824>

If you can't connect to your Checkpoint VPN server you can try using other SNX builds.

If the container started up you could quickly test the new SNX build as follows:

1. Copy the SNX build from the docker host to the docker container

    ```sh
    docker cp snx_install.sh snx-vpn:/
    ```

2. Connect to the docker container

    ```sh
    docker exec -ti snx-vpn bash
    ```

3. Get process ID of the currently running SNX client (if any):

    ```sh
    ps ax
    ```

    Expected output similar to:

    ```sh
    PID TTY      STAT   TIME COMMAND
        1 pts/0    Ss     0:00 /bin/bash /root/snx.sh
    29 ?        Ss     0:00 snx -s ip_vpn_server -c /certificate.p12
    32 pts/0    S+     0:00 /bin/bash
    37 pts/1    Ss     0:00 bash
    47 pts/1    R+     0:00 ps ax
    ```

4. Kill the process (in this example 29):

    ```sh
    kill 29
    ```

5. Adjust permissions of the SNX build

    ```sh
    chmod a+rx snx_install.sh
    ```

6. Execute the installation file:

    ```sh
    ./snx_install.sh
    ```

    Expected output:

    ```sh
    Installation successfull
    ```

7. Check installation:

    ```sh
    ldd /usr/bin/snx
    ```

    Expected output similar to:

    ```sh
    linux-gate.so.1 (0xf7f3c000)
    libX11.so.6 => /usr/lib/i386-linux-gnu/libX11.so.6 (0xf7dea000)
    libpthread.so.0 => /lib/i386-linux-gnu/libpthread.so.0 (0xf7dcb000)
    libresolv.so.2 => /lib/i386-linux-gnu/libresolv.so.2 (0xf7db3000)
    libdl.so.2 => /lib/i386-linux-gnu/libdl.so.2 (0xf7dae000)
    libpam.so.0 => /lib/i386-linux-gnu/libpam.so.0 (0xf7d9e000)
    libnsl.so.1 => /lib/i386-linux-gnu/libnsl.so.1 (0xf7d83000)
    libstdc++.so.5 => /usr/lib/i386-linux-gnu/libstdc++.so.5 (0xf7cc4000)
    libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xf7ae8000)
    libxcb.so.1 => /usr/lib/i386-linux-gnu/libxcb.so.1 (0xf7abc000)
    /lib/ld-linux.so.2 (0xf7f3e000)
    libaudit.so.1 => /lib/i386-linux-gnu/libaudit.so.1 (0xf7a92000)
    libm.so.6 => /lib/i386-linux-gnu/libm.so.6 (0xf798e000)
    libgcc_s.so.1 => /lib/i386-linux-gnu/libgcc_s.so.1 (0xf7970000)
    libXau.so.6 => /usr/lib/i386-linux-gnu/libXau.so.6 (0xf796c000)
    libXdmcp.so.6 => /usr/lib/i386-linux-gnu/libXdmcp.so.6 (0xf7965000)
    libcap-ng.so.0 => /lib/i386-linux-gnu/libcap-ng.so.0 (0xf795f000)
    libbsd.so.0 => /lib/i386-linux-gnu/libbsd.so.0 (0xf7944000)
    librt.so.1 => /lib/i386-linux-gnu/librt.so.1 (0xf793a000)
    ```

8. Manually try to connect:

    ```sh
    snx -s ip_vpn_server -c /certificate.p12
    ```

    Expected output similar to:

    ```sh
    Please enter the certificate's password:
    ```

9. Type the password and press <kbd>Enter</kbd>

    Expected output similar to:

    ```sh
    SNX - connected.

    Session parameters:
    ===================
    Office Mode IP      : 192.168.90.82
    DNS Server          : 10.20.30.41
    Secondary DNS Server: 10.20.30.42
    Timeout             : 6hours 
    ```

## Make the connection easier

After you created the container and checked that the SNX client works, you could create a script to make the whole process easier:

Example script `snx-vpn.sh` that starts the container and automatically adds a route:

```sh
#! /bin/bash
COMMAND_PARAM=${1:-}
if [ "$COMMAND_PARAM" != "start" ] && [ "$COMMAND_PARAM" != "stop" ]; then
    echo -e "'$COMMAND_PARAM' is not a valid command!"
    exit 1
fi

if [ "$COMMAND_PARAM" == "start" ]; then
    IP=$2
    if [ -z "$IP" ]; then
        echo "IP argument required"
        exit 1
    fi
    NET_MASK=${3:-255.255.255.255}
    SNX_DOCKER_NAME=${4:-snx-vpn}
    IS_DOCKER_RUNNING=$(docker inspect -f '{{ .State.Running }}' "$SNX_DOCKER_NAME")
    if [ "true" == "$IS_DOCKER_RUNNING" ]; then
        exit 0
    fi

    docker start "$SNX_DOCKER_NAME"
    SNX_DOCKER_IP=$(docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$SNX_DOCKER_NAME")
    echo "adding route -net $IP netmask $NET_MASK gw $SNX_DOCKER_IP"
    sudo route add -net "$IP" netmask "$NET_MASK" gw "$SNX_DOCKER_IP"
    exit 0
fi

if [ "$COMMAND_PARAM" == "stop" ]; then
    SNX_DOCKER_NAME=${2:-snx-vpn}
    IS_DOCKER_RUNNING=$(docker inspect -f '{{ .State.Running }}' "$SNX_DOCKER_NAME")
    if [ "true" == "$IS_DOCKER_RUNNING" ]; then
        docker stop "$SNX_DOCKER_NAME"
        exit 0
    fi
fi
```

Usage:

```sh
./snx-vpn.sh start <ip> [<netmask>] [<container-name>]
```

```sh
./snx-vpn.sh stop [<container-name>]
```

`<ip>`: required, the IP address you want to reach behind the VPN.

`<netmask>`: optional, defaults to `255.255.255.255`.

`<container-name>`: optional, defaults to `snx-vpn`.

## Credits

This image is inspired by the excellent work of:

<https://github.com/Kedu-SCCL/docker-snx-checkpoint-vpn>

<https://github.com/iwanttobefreak/docker-snx-vpn>

<https://github.com/mnasiadka/docker-snx-dante>

<https://unix.stackexchange.com/a/453727>

<https://gitlab.com/jamgo/docker-juniper-vpn>
