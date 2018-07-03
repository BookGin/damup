# Override DNS Record

We have 3 options to override the DNS record.

1. Server side: delegate `*.example.com` to a customized DNS server
  - This is the easiest approach.
2. Server side: customized DNS server, Client side: use remote DNS of SOCKS5 proxy
  - The client uses remote DNS of SOCKS5 proxy to resolve `*.example.com`.
  - The server build a SOCKS5 proxy server and a local DNS server to resolve the addresses.
3. Client side: overwrite `/etc/hosts` file in client side
  - The approach requires root permission, and it's not very flexible since it doesn't support wildcard domains.

## 1. Delegation

- Insert a `NS` record pointing to the customized DNS server.
- The DNS server resolves wildcard addresses  `*.example.com` to the IP address of Proxy.

## 2. Remote DNS with SOCKS5

### Install SOCKS5 Server

We use [Dante](https://www.inet.no/dante/) as the SOCKS5 server. Download the source code and compile.

```sh
wget https://www.inet.no/dante/files/dante-1.4.2.tar.gz
tar zxvf dante-1.4.2.tar.gz
cd dante-1.4.2
./configure
make -j`nproc`
make install
```

### SOCKS5 Server Configuration

- `PROXY_PUBLIC_IP`: proxy's public IP address for the client to connect
- `PROXY_INTERNAL_IP_TO_ORIGIN_SERVER`: proxy's internal IP address, which is used to access origin server via the secure tunnel
- `ORIGIN_SERVER_IP`: origin server IP address

```
# sockd.conf
# Dante SOCKS5 proxy configuration file

# logoutput: /dev/stderr
# debug: 1

internal: PROXY_PUBLIC_IP port = 4500
external: PROXY_INTERNAL_IP_TO_ORIGIN_SERVER

clientmethod: none
socksmethod: none
user.notprivileged: nobody

socks pass {  
        from: 0.0.0.0/0 to: ORIGIN_SERVER_IP/32
        log: error error connect disconnect
}

client pass {
        from: 0.0.0.0/0 to: ORIGIN_SERVER_IP/32
        log: error connect disconnect
}
```

Then, start the SOCKS5 server:

```sh
# Execute this command as root, and it will run as "nobody"
$ ./sbin/sockd -f sockd.conf
```

### Client Proxy Configuration

We use [Proxy Auto-Configuration (PAC) file](https://developer.mozilla.org/en-US/docs/Web/HTTP/Proxy_servers_and_tunneling/Proxy_Auto-Configuration_(PAC)_file) such that client will only proxy the request when visiting `*.example.com`.  The file can be either in the local file system or on the Internet.

```
function FindProxyForURL(url, host) {
  if (dnsDomainIs(host,"example.com"))
    return "SOCKS5 DAMUP_PROXY_HOSTNAME:4500";
  return "DIRECT";                                                              
}
```

Don't forget to enable SOCKS 5 remote DNS resolution.

## 3. Overwrite /etc/hosts

Append this line in the `/etc/hosts`:

```
PROXY_IP SNI_KEY.example.com
```