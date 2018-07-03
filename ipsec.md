# IPsec Secure Tunnel

We'll establish a secure IPsec tunnel between Gateway and Proxy using [strongSwan](https://www.strongswan.org/). The Proxy will access the origin server with IPsec VPN via Gateway.

For simplicity's sake, the authentication is based on pre-shared secret (PSK). strongSwan supports a number of [other authentication methods](https://wiki.strongswan.org/projects/strongswan/wiki/IpsecSecrets).

This chapter mainly refers to the [official test suite](https://www.strongswan.org/testing/testresults/ikev2/rw-psk-ipv4/index.html).

## Install strongSwan 

Ubuntu 18.04 Bionic Beaver: strongSwan 5.6.2

```sh
apt install strongswan
```

Instead of installing from the package, you can also compile from the [source](https://www.strongswan.org/).

## Gateway

### strongSwan Configuration
```
# /etc/ipsec.secrets - strongSwan IPsec secrets file

PROXY_IP : PSK "SECRET_PASSWORD"
```

```
# /etc/ipsec.conf - strongSwan IPsec configuration file

config setup

conn %default
	ikelifetime=60m
	keylife=20m
	rekeymargin=3m
	keyingtries=1
	keyexchange=ikev2
	authby=secret

conn rw
	left=GATEWAY_IP
	leftsubnet=ORIGIN_SERVER_IP/32
	leftfirewall=yes
	right=PROXY_IP
	auto=add
```
### Notes

1. The Gateway will also need to set up the firewall rules and enable traffic forwarding. Please refer to [the official documentation](https://wiki.strongswan.org/projects/strongswan/wiki/ForwardingAndSplitTunneling).
2. The `ORIGIN_SERVER_IP` should be a private IP. Otherwise, a firewall rules should be set to block all the direct access traffic.

## Proxy

### strongSwan Configuration
```
# /etc/ipsec.secrets - strongSwan IPsec secrets file

PROXY_IP : PSK "SECRET_PASSWORD"
```

```
# /etc/ipsec.conf - strongSwan IPsec configuration file

config setup

conn %default
	ikelifetime=60m
	keylife=20m
	rekeymargin=3m
	keyingtries=1
	keyexchange=ikev2
	authby=secret

conn home
	left=PROXY_IP
	leftfirewall=yes
	right=GATEWAY_IP
	rightsubnet=ORIGIN_SERVER_IP/32
	auto=add
```

### Notes

1. The Proxy is actually a VPN client connecting to the server Gateway. To establish the connection:

   ```sh
   $ ipsec restart
   $ ipsec up home
   ```

2. If the connection is established successfully,  you will be able to ping `ORIGIN_SERVER_IP`.
