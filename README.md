# Docker SNX Checkpoint VPN

## Introduction

Client for Checkpoint VPN using snx GNU/Linux client. It accepts username and/or certificate.

## Usage

### 1. Create the container

* With docker-compose (`.p12` certificate file required)

    1. Create a `.env` file

        Create a `.env` file in the root of the project, alongside `docker-compose.yml`, with your env variables (take a look at `.env.example`).

    2. Start the container

        ```bash
        docker-compose up -d
        ```

OR

* With docker

    1. With certificate

        ```bash
        docker run --name snx-vpn \
        --cap-add=ALL \
        -v /lib/modules:/lib/modules \
        -e SNX_SERVER=vpn_server_ip_address \
        -e SNX_PASSWORD=secret \
        -v /path/to/my_snx_vpn_certificate.p12:/certificate.p12 \
        -t \
        -d ananni/snx-checkpoint-vpn
        ```

        **IMPORTANT**: specify a volume with `/certificate.p12` as container path

    2. With certificate and username

        ```bash
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

        **IMPORTANT**: specify a volume with `/certificate.p12` as container path

    3. Without certificate

        ```bash
        docker run --name snx-vpn \
        --cap-add=ALL \
        -v /lib/modules:/lib/modules \
        -e SNX_SERVER=vpn_server_ip_address \
        -e SNX_USER=user \
        -e SNX_PASSWORD=secret \
        -t \
        -d ananni/snx-checkpoint-vpn
        ```

### 2. Get IP address of the container

```bash
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' snx-vpn
```

Example output:

```bash
172.17.0.2
```

### 3. Add a route using the container IP address as gateway

```bash
sudo route add -net 10.20.30.0 gw 172.17.0.2 netmask 255.255.255.0
```

If netmask `255.255.255.0` isn't working, you can try `255.255.255.255`.

### 4. Try to reach the server behind SNX VPN (in this example through SSH)

```bash
ssh 10.20.30.40
```

## Environment Variables

`SNX_SERVER`: Mandatory. IP address or name of the Checkpoint VPN server

`SNX_PASSWORD`: Mandatory. String corresponding to the password of VPN client

`SNX_USER`: Optional if certificate volume has been provided, otherwise mandatory. String corresponding to the username of VPN client

With docker-compose you also need the following variables:

`CONTAINER_NAME`: Name to assign to the container

`LOCAL_CERTIFICATE_PATH`: Local absolute path to your `.p12` certificate

## Allowed volumes

`/certificate.p12`: A VPN client certificate. If present the SNX binary will be invoked with `-c` parameter pointing to this certificate file.

## Routes

Since the container is the one that connects to the VPN server, it's the one that receives the routes. In order to list all of them perform the following command from the docker host (`snx-vpn` is the container name in this example):

```bash
docker exec -ti snx-vpn route -n | grep -v eth0
```

Expected output similar to:

```bash
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.20.30.0       0.0.0.0         255.255.255.0   U     0      0        0 tunsnx
10.20.40.0       0.0.0.0         255.255.255.0   U     0      0        0 tunsnx
...
```

Then manually add a route:

```bash
sudo route add -net 10.20.30.0 netmask 255.255.255.0 gw `docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' snx-vpn`
```

And finally test access. In this example trying to reach via SSH a remote server:

```bash
ssh user@10.20.30.40
```

## DNS

Since the container is the one that connects to the VPN server, it's the one that receives the DNS servers. In order to get them you can proceed in on of two ways:

* Check the container logs (`snx-vpn` is the container name in this example)

    ```bash
    docker logs snx-vpn | grep DNS
    ```

    Expected output similar to:

    ```bash
    DNS Server          : 10.20.30.11
    Secondary DNS Server: 10.20.30.12
    ```

OR

* Check `/etc/resolv.conf` container file (`snx-vpn` is the container name in this example)

    ```bash
    docker exec -ti snx-vpn cat /etc/resolv.conf
    ```

    Expected output similar to:

    ```bash
    nameserver 10.20.30.11
    nameserver 10.20.30.12
    nameserver 8.8.4.4
    nameserver 8.8.8.8
    ```

Once you know the DNS servers you can proceed in one of two ways:

* Update your docker host `/etc/resolv.conf`

    ```bash
    sudo vim /etc/resolv.conf
    ```

    With below content:

    ```bash
    nameserver 10.20.30.11
    nameserver 10.20.30.12
    ```

    You should remember to revert back the changes once finished

OR

* Run a local dnsmasq service. It requeries that you know the remote domains beforehand (`example.com` in this example)

    1. Create the file:

        ```bash
        sudo vim /etc/dnsmasq.d/example.com
        ```

        With below content:

        ```bash
        server=/example.com/10.20.30.11
        ```

    2. Restart the "dnsmasq" service

        ```bash
        sudo service dnsmasq restart
        ```

    3. Test it

        ```bash
        ssh server.example.com
        ```

## Troubleshooting

This image has just been tested without username and with certificate, and with snx build 800010003 obtained from:

<https://supportcenter.checkpoint.com/supportcenter/portal/user/anon/page/default.psml/media-type/html?action=portlets.DCFileAction&eventSubmit_doGetdcdetails=&fileid=22824>

If you can't connect to your Checkpoint VPN server you can try using other SNX builds.

If the container started up you could quickly test the new SNX build as follows:

1. Copy the SNX build from the docker host to the docker container

    ```bash
    docker cp snx_install.sh snx-vpn:/
    ```

2. Connect to the docker container

    ```bash
    docker exec -ti snx-vpn bash
    ```

3. Get process ID of the currently running SNX client (if any):

    ```bash
    ps ax
    ```

    Expected output similar to:

    ```bash
    PID TTY      STAT   TIME COMMAND
        1 pts/0    Ss     0:00 /bin/bash /root/snx.sh
    29 ?        Ss     0:00 snx -s ip_vpn_server -c /certificate.p12
    32 pts/0    S+     0:00 /bin/bash
    37 pts/1    Ss     0:00 bash
    47 pts/1    R+     0:00 ps ax
    ```

4. Kill the process (in this example 29):

    ```bash
    kill 29
    ```

5. Adjust permissions of the SNX build

    ```bash
    chmod a+rx snx_install.sh
    ```

6. Execute the installation file:

    ```bash
    ./snx_install.sh
    ```

    Expected output:

    ```bash
    Installation successfull
    ```

7. Check installation:

    ```bash
    ldd /usr/bin/snx
    ```

    Expected output similar to:

    ```bash
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

    ```bash
    snx -s ip_vpn_server -c /certificate.p12
    ```

    Expected output similar to:

    ```bash
    Please enter the certificate's password:
    ```

9. Type the password and press <kbd>Enter</kbd>

    Expected output similar to:

    ```bash
    SNX - connected.

    Session parameters:
    ===================
    Office Mode IP      : 192.168.90.82
    DNS Server          : 10.20.30.41
    Secondary DNS Server: 10.20.30.42
    Timeout             : 6hours 
    ```

## Make the connection easier

Once you checked that the SNX client works, you could create a script to make the whole process easier:

1. Create the script

    ```bash
    sudo vim /usr/local/bin/snx-vpn.sh
    ```

    With below content, adjusting `SNX_DOCKER_NAME` and routes to match your needs:

    ```bash
    #! /bin/bash
    SNX_DOCKER_NAME="snx-vpn"
    IS_DOCKER_RUNNING="$(docker inspect -f '{{ .State.Running }}' $SNX_DOCKER_NAME)"
    if [ "true" == $IS_DOCKER_RUNNING ]; then
        exit 0
    fi
    docker start $SNX_DOCKER_NAME
    SNX_DOCKER_IP="$(docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $SNX_DOCKER_NAME)"
    # Add custom rules behind this line
    #sudo route add -net 10.20.30.0 netmask 255.255.255.0 gw $SNX_DOCKER_IP
    ```

2. Make it executable

    ```bash
    chmod +x /usr/local/bin/snx-vpn.sh
    ```

3. Test it:

    ```bash
    snx-vpn.sh
    ```

## Credits

This image is inspired by the excellent work of:

<https://github.com/Kedu-SCCL/docker-snx-checkpoint-vpn>

<https://github.com/iwanttobefreak/docker-snx-vpn>

<https://github.com/mnasiadka/docker-snx-dante>

<https://unix.stackexchange.com/a/453727>

<https://gitlab.com/jamgo/docker-juniper-vpn>
