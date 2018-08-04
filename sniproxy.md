# SNI Proxy

The SNI Proxy server is installed on the Proxy. It will compute the HMAC-SHA256 of the Client ID and UNIX timestamp in the hostname. 

The hostname should consist of Client ID , UNIX timestamp, and base32 encoded HMAC-SHA256.

For example, this is a valid format:

`jackhammer.1533292236.3gdzbmwfy0y3nbdp4hkuozop33k435ie1244fwqq22lr5qjil55a.example.com`

- Client ID: `jackhammer`
- UNIX timestamp (the token is valid until): 1533292236
- base32 encoded HMAC-SHA256 of `jackhammer.1533292236`: `3gdzbmwfy0y3nbdp4hkuozop33k435ie1244fwqq22lr5qjil55a`

## Design

### Client ID

The Client ID consists of case-insensitive alphanumeric characters. The length should be less than 64 bytes.

Possible issues:
- ISP or a middlebox can leverage the Client ID/HMAC signature to trace the user.
- A possible solution is to rotate the Client ID every T seconds.

### Valid-Until UNIX Timestamp

The token is expired after the UNIX timestamp.

### Base32 Encoded HMAC-SHA256 Signature

According to [RFC 952 and 1123](https://stackoverflow.com/a/3523068), only case-insensitive alphanumeric characters and hyphen are valid in the hostname. Therefore, the implementation is based on revised base32(https://en.wikipedia.org/wiki/Base32). The alphabet used is `abcdefghijklmnopqrstuvwxyz012345`. There is no padding alphabet in the implementation, because the size of HMAC-SHA256 is always 256 bits. 

The size of base32 encoded HMAC-SHA256 signature is 52 bytes.

## Install SNI Proxy

We use Dustin Lundquist's [dlundquist/sniproxy](https://github.com/dlundquist/sniproxy) as the implementation. 

The program itself does not support signature verification. I modify the source code to implement the feature. You can check out the [diff](https://github.com/dlundquist/sniproxy/compare/master...BookGin:master) on GitHub.

```sh
git clone https://github.com/bookgin/sniproxy
cd sniproxy
# You can safely ignore the error: debchange: command not found
./autogen.sh
# HMAC requires linking to OpenSSL library
env CFLAGS='-lcrypto' ./configure --prefix=INSTALL_PREFIX
make -j`nproc`
make install
```

To run the server in foreground:

```sh
./sniproxy -f -c sniproxy.conf
```
## Configuration

sniproxy.conf:

```
user nobody

pidfile /tmp/sniproxy.pid

error_log {
    filename ./error_log
    priority debug
}

access_log {
    filename ./access_log
}

listener 127.0.0.1:443 {
    protocol tls
    table damup
}

table damup {
    HMAC-SHA256-SECRET-KEY 192.168.1.1:443
}
```

- Replace the `192.168.1.1:8080` with the internal IP address of the origin server.
- Only the table named `damup` will implement DAMUP features.
- In the `damup` table, it should only be one entry. Other entries are simply ignored.
- The entry should contains the HMAC-SHA256 secret key.
- Currently, the secret key should only contain alphanumeric characters and hyphen.
- The fallback address can also be used, which is discussed in the paper Section V (A)-2.

The configuration man page can be found [here](https://github.com/dlundquist/sniproxy/blob/master/man/sniproxy.conf.5).
