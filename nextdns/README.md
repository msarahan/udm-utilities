# Run NextDNS on your UDM

### Features
1. Run NextDNS on your UDM with a completely isolated network stack.  This will not port conflict or be influenced by any changes on by Ubiquiti.
2. Resolves IP addresses handed out by DHCP on the UDM!
3. Persists through reboots and firmware updates.

### Requirements
1. You have already setup the on boot script described [here](https://github.com/boostchicken/udmpro-utilities/tree/master/on-boot-script)
2. NextDNS persists through firmware updates. The on-boot script does not.  If you update your FM setup on-boot again and everything should work.

### Customization
* Feel free to change [20-dns.conflist](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/20-dns.conflist) to change the IP address of the container. Make sure to update all ip references and the iptables rules in [on_boot.sh](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/on_boot.sh).  The IP address can be anywhere from x.x.x.3 to x.x.x.254. .1 is reserved for the gateway and .2 is reserved for the macvlan bridge.
* If you don't want to use vlan5, just replace br5 with br(vlanid) in [on_boot.sh](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/on_boot.sh) and [20-dns.conflist](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/20-dns.conflist), also update all the ips accordingly
* The NextDNS docker image is not supported by NextDNS. It is built out of this repo.  If you make any enhancements please constribute back via a Pull Request.

### Docker
The Dockerfile is included, you can build it locally on your UDM if you don't want to pull from Docker Hub or make customizations
```
podman build . -t nextdns-udm:latest"
```
Building from another device is possible.  You must have [buildx](https://github.com/docker/buildx/) installed to do cross platform builds. This is useful if you want to mirror to a private repo
```
docker buildx build --platform linux/arm64 -t nextdns-udm:latest .
```

### Steps
1. On your controller, make a Corporate network with no DHCP server and give it a vlan. For this example we are using vlan 5.
2. Install the CNI plugins with by executing [install-cni-plugins.sh](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/install-cni-plugins.sh) on your UDM
3. Copy [20-dns.conflist](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/20-dns.conflist) to /mnt/data/podman/cni (or a place of your choosing and update [on_boot.sh](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/on_boot.sh) symlink).  This will create your podman macvlan network
4. Update your on_boot.sh to include the commands in [on_boot.sh](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/on_boot.sh).  You can leave out the iptables rules if you don't want to DNAT all DNS calls to your PiHole
5. Execute on_boot.sh
6. Make /mnt/data/nextdns and copy [nextdns.conf](https://github.com/boostchicken/udm-utilities/blob/master/nextdns/udm-files/nextdns.conf) to it.
7. Run the NextDNS docker container.  The mounts are very important. Also, please change the --dns arguments to whatever was provided by NextDNS.

```
 podman run -d --privileged --network dns \
    --name nextdns \
    -v "/mnt/data/nextdns/:/etc/nextdns/" \
    -v /var/run/dbus/system_bus_socket:/var/run/dbus/system_bus_socket \
    --mount type=bind,source=/config/dnsmasq.lease,target=/tmp/dnsmasq.leases \
    --dns=45.90.28.163 --dns=45.90.30.163 \
    --hostname nextdns \
    boostchicken/nextdns-udm:latest
```
Note: Mounting dbus and running in privileged is not required.  It's for mDNS.
 
8. Update your DNS Servers to 10.0.5.3 (or your custom ip) in all your DHCP configs.