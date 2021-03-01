# docker-vpn-ap
Let all devices connected to Wi-Fi access point share the VPN connection

This project can provide these nice features:
- It supports almost all VPN service. So long as it has docker images provided and the tunnel interface can be found(e.g. "tun0")
- Only one VPN license/qutota is needed for the VPN container
- All devices conencted to the AP can share the VPN connection
- Only VPN/AP container connect to the VPN tunnel while other services wouldn't connect to it
- Support DNS config for the AP. Can be integrate with DNS servers, like Pi-hole, AdGuard, etc
- Allow to enable / disable local network connection

It can be run for Linux device. I tested it in 64 bit Raspbian & Ubuntu

Noted that for your raspberry pi 4B which runs Raspbain, the driver of the default WiFi chip has some issue right now for AP mode. You need to apply the patch the WiFi driver:
https://github.com/raspberrypi/linux/issues/3619

## To setup your own VPN AP:
```
git clone https://github.com/dannypv05261/docker-vpn-ap.git

# replace 'surfshark' service with your own vpn service and make sure 'privileged': true is added to that service
# replace AP info: DRIVER, INTERFACE, WPA_PASSPHRASE, SSID, etc
vim docker-compose.yml

# Make sure hostapd config is fit for your WiFi chips
vim dokcer/startwlan.sh

# Enable IP forwarding for AP
sysctl -w net.ipv4.ip_forward=1

docker-compose build
docker-compose up
```

If you can't catch up the above steps. Please try to flow the step-by-step tutorial below.

## Step by Step tutorial:
### Step 1: Make sure you have one network interface for the Internet connection & another network interface for AP

You can check your network interface with this command
```
ifconfig

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        .......
        RX packets 86469  bytes 21402997 (20.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 65549  bytes 11962406 (11.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
wlan0: ......

wlan1: ......
```
I pick eth0 (LAN port) as the Internet connection (You can also use WiFi interface, like wlanX) and wlan0 for AP.

**Make sure your WiFi chipset for AP interface support AP mode (pi 3B/4B default WiFi chipset supports it)**
```sh
iw list

# if it has sth like #{ AP } with value larger than 1. It means that it supports AP mode
# valid interface combinations:
#                 * #{ managed } <= 1, #{ P2P-device } <= 1, #{ P2P-client, P2P-GO } <= 1,
#                   total <= 3, #channels <= 2
#                 * #{ managed } <= 1, #{ AP } <= 1, #{ P2P-client } <= 1, #{ P2P-device } <= 1,
#                   total <= 4, #channels <= 1
```


### Step 2: Prepare your own VPN docker images
In my example, I use surfshark in docker-compose.yml. 
You need to create your own docker-compose.yml according to your VPN docker image chosen.
Noted that 'privileged: true' in your VPN config is needed for AP container to manipulate the AP device and AP container will depend on your VPN container. Please add it if the config you choose is missing it.
```
# Start: Replace this block with your VPN servie provider
    # Noted that 'privileged: true' in your VPN config is needed for docker-ap to manipulate AP
    surfshark:
        image: ilteoood/docker-surfshark
        container_name: surfshark
        environment: 
            - SURFSHARK_USER=YOUR_USERNAME
            - SURFSHARK_PASSWORD=YOUR_PASSWORD
            - SURFSHARK_COUNTRY=jp
            - SURFSHARK_CITY=tok
            - CONNECTION_TYPE=udp
            - LAN_NETWORK=192.168.1.0/24
        cap_add: 
            - NET_ADMIN
        devices:
            - /dev/net/tun
        privileged: true
        restart: unless-stopped
    # End:: Replace this block with your VPN servie provider
```

Run your VPN container and make sure it is healthy
```sh
docker-compose up
```

### Step 3: Check VPN tunnel interface in the VPN container
Assume your VPN container is running sucessfully. You need to make sure tunnel interface (e.g. tun0) can be found
```sh
# find your docker container ID
docker ps

# Check network interface of the VPN container (cc858c03c241 should be replaced by your onw docker container ID)
docker exec cc858c03c241 ifconfig
#tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
#          inet addr:x.x.x.x  P-t-P:x.x.x.x  Mask:x.x.x.x
#          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
#          RX packets:5769 errors:0 dropped:0 overruns:0 frame:0
#          TX packets:5814 errors:0 dropped:0 overruns:0 carrier:0
#          collisions:0 txqueuelen:500
#          RX bytes:1866210 (1.7 MiB)  TX bytes:1063854 (1.0 MiB)
```

### Step 4: Enable IP forwarding in your host machine
```
# For Rasbian & Ubuntu

# Check status
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1

# Enable it if it is 0 
sysctl -w net.ipv4.ip_forward=1
```

### Step 5: Add AP config to your docker-compose.yml
I don't want to publish the docker images, so you need to download the whole docker-ap folder and place the folder next to your docker-compose.yml

Now edit your docker-compose.yml and parse the following infor into it
You need to add 'network_mode' and 'depends_on' point to your VPN container
```
    docker-ap:
      build:
        context: docker-ap
        dockerfile: Dockerfile
      container_name: docker-ap
      image: dokcer-ap
      restart: unless-stopped
      privileged: true
      depends_on: 
        surfshark:  # You need to edit this link to your VPN container service name
          condition: service_healthy #Make sure the AP container starts after your VPN container becomes healthy
      network_mode: service:surfshark   # You need to edit this link to your VPN container service name
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock 
      environment:
        - INTERFACE=wlan0 # the interface name for AP
        - OUTGOINGS=tun0,eth0  #tun0 is for Internet access via VPN and eth0 is for LAN access. You can remove 'eth0' if you don't want the device connected to the AP can access your LAN network
        - CHANNEL=36
        - WPA_PASSPHRASE=password  #Your AP password
        - SSID=docker-ap           #Your AP SSID
        - SUBNET=192.168.2.0
        - AP_ADDR=192.168.2.1
        - DNS_ADDRESSES=192.168.1.1 #e.g. 1.1.1.1,1.0.0.1    You change the DNS of your AP to sth like route/firewall/ Pi-hole/AdGuard
```

Noted that the current AP config is based on 802.11ac. If you want to use 802.11n, you can refer to https://github.com/offlinehacker/docker-ap and change hostapd config based on his wlanstart.sh file

### Step 6: Start the service and connect to your SSID after AP container is healthy
```
docker-compose down && docker-compose up
docker-compose down && docker-compose up -d
```

# Use multiple AP with different SSID for VPN connected to different Country
You can simply copy the existing docker-compose.yml, change the VPN info, and run docker-compose -f docker-compose-tw.yml to specify the config file.

However, for most of the USB with AP, it only supports 1 AP mode. i.e. #{ AP } <= 1.
In this case, you need to use another USB WiFi dongle for another AP.

Some of the WiFi chip supports multiple AP. e.g. #{ AP } <= 8.
You may not need to buy additional USB WiFi dongle, but you need to apply the change according to this link http://wiki.stocksy.co.uk/wiki/Multiple_SSIDs_with_hostapd
```sh
iw list

# if it has sth like #{ AP } with value larger than 1. It means that it supports AP mode
# valid interface combinations:
#                 * #{ managed } <= 1, #{ P2P-device } <= 1, #{ P2P-client, P2P-GO } <= 1,
#                   total <= 3, #channels <= 2
#                 * #{ managed } <= 1, #{ AP } <= 1, #{ P2P-client } <= 1, #{ P2P-device } <= 1,
#                   total <= 4, #channels <= 1
```

# Other info: Other docker containers also shared the VPN connection
You can add that service into the same docker-compose file and add this line to the config of that service: network_mode: service:${your vpn container name}

# Other info: The host machine also want to use VPN connection
You can simply change the network mode to host in your VPN container by network_mode: host

# Reference
This project is based on https://github.com/offlinehacker/docker-ap

https://github.com/ilteoood/docker-surfshark
