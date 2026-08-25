You bring up an excellent point, and you are exactly right when it comes to **standard** PSK.

In a basic PSK cipher suite (like `PSK-AES256-GCM-SHA384`), the pre-shared key is used directly to derive the session keys. Because there is no Diffie-Hellman exchange, there is **no Perfect Forward Secrecy (PFS)**. If someone compromises your PSK in the future, they can decrypt all past recorded traffic.

However, TLS (and by extension, OpenSSL and stunnel) supports **ECDHE-PSK** (Elliptic Curve Diffie-Hellman Ephemeral with PSK).

When you use an `ECDHE-PSK` cipher suite, the roles are split:

1. **Diffie-Hellman (ECDHE)** handles the actual key exchange, guaranteeing Perfect Forward Secrecy.
2. **The Pre-Shared Key (PSK)** is used strictly for **authentication** (proving the client and server know the secret password), replacing the need for a certificate.

Here is how you configure stunnel to use zero certificates while strictly enforcing Diffie-Hellman for Forward Secrecy via PSK.

---

### 1. Create the PSK Secrets File

Create a plain text file on **both** the client and the server containing your identity and the shared secret.

```bash
# Create /etc/stunnel/psk.txt
echo "my-stunnel-client:YourVeryLongAndSecureRandomPasswordHere" > /etc/stunnel/psk.txt

# Secure the file
chmod 600 /etc/stunnel/psk.txt

```

### 2. Server Configuration (`stunnel.conf`)

Notice that all `cert` and `verifyChain` directives are gone. We replace them with `PSKsecrets` and restrict the `ciphers` to only allow ECDHE-PSK combinations.

```ini
; /etc/stunnel/stunnel.conf (Server)

foreground = no
pid = /var/run/stunnel.pid

[my-secure-service]
accept = 0.0.0.0:8443
connect = 127.0.0.1:8080

; Point to the PSK file
PSKsecrets = /etc/stunnel/psk.txt

; Force TLS 1.2 (or higher)
sslVersion = TLSv1.2

; Strictly enforce ECDHE-PSK cipher suites to guarantee Forward Secrecy
ciphers = ECDHE-PSK-AES256-GCM-SHA384:ECDHE-PSK-CHACHA20-POLY1305

```

### 3. Client Configuration (`stunnel.conf`)

The client configuration mirrors the server. It uses the exact same secrets file to authenticate itself to the server.

```ini
; /etc/stunnel/stunnel.conf (Client)

foreground = no
pid = /var/run/stunnel-client.pid
client = yes

[my-secure-client]
accept = 127.0.0.1:8081
connect = server_ip_address:8443

; Point to the PSK file and declare the client identity
PSKsecrets = /etc/stunnel/psk.txt
PSKidentity = my-stunnel-client

; Force TLS 1.2 (or higher)
sslVersion = TLSv1.2

; Enforce the same ECDHE-PSK cipher suites
ciphers = ECDHE-PSK-AES256-GCM-SHA384:ECDHE-PSK-CHACHA20-POLY1305

```

By forcing `ECDHE-PSK-AES256-GCM-SHA384`, you get exactly what you are looking for: zero certificates, mutual authentication via a shared password, and ephemeral Diffie-Hellman math ensuring your tunnel has Perfect Forward Secrecy.

Yes, `stunnel` can absolutely do this, but it is important to understand exactly what data your script will be handling.

### 1. How `exec` works with `client = yes`

When you configure `stunnel` as a client (`client = yes`) and use `exec` instead of `connect`, here is the data flow:

1. `stunnel` accepts a **plaintext** connection on the `accept` socket.
2. `stunnel` encrypts that plaintext data into TLS records.
3. `stunnel` executes your script and pipes the **encrypted TLS data** to the script's Standard Input (`stdin`).
4. `stunnel` listens to your script's Standard Output (`stdout`) and expects it to return **encrypted TLS data**, which it will then decrypt and send back to the original local socket.

This is typically used when you want a custom transport layer for your TLS traffic (like piping the TLS stream through `ssh`, a serial port, or a custom UDP wrapper).

### 2. What `REMOTE_HOST` and `REMOTE_PORT` contain

In an `exec` environment, `REMOTE_HOST` and `REMOTE_PORT` contain the **source IP and source port** of the client that connected to the `accept` socket.

Because `stunnel` acts as the middleman triggering the script, it considers the entity that initiated the TCP connection to be the "remote" party.

* **`REMOTE_HOST`**: The IP address of the local application that just connected to your `accept` socket (often `127.0.0.1` if it's local).
* **`REMOTE_PORT`**: The ephemeral source port used by that application to make the connection (e.g., `54321`).

### Example Configuration

Here is how that looks in your `stunnel.conf`:

```ini
[custom-transport-client]
client = yes
accept = 127.0.0.1:8081

; Execute your custom script instead of connecting to a remote socket
exec = /usr/local/bin/my_transport_script.sh

; (Optional) Pass arguments to your script
execArgs = my_transport_script.sh arg1 arg2

```

**Note:** If your goal was actually for `stunnel` to accept an *encrypted* connection, decrypt it, and pass *plaintext* to your script, you need to run `stunnel` in server mode (`client = no`).

```
#!/usr/bin/env bash

get_local_ip() {
  case "${OSTYPE}" in
    linux-gnu*)
      # iproute2 is the native Linux standard. 
      # We scan for 'src' to avoid hardcoding awk columns that might shift.
      ip route get 8.8.8.8 2>/dev/null | awk '{for(i=1;i<=NF;i++) if($i=="src") print $(i+1)}'
      ;;
    darwin*)
      # macOS uses BSD-style routing. We extract the active interface first.
      local iface
      iface=$(route get 8.8.8.8 2>/dev/null | awk '/interface:/ {print $2}')
      
      # Then use the native macOS tool to get that specific interface's IP
      if [[ -n "$iface" ]]; then
        ipconfig getifaddr "${iface}"
      fi
      ;;
    *)
      echo "Error: Unsupported OS for native IP detection" >&2
      return 1
      ;;
  esac
}

# Usage
ACTIVE_IP=$(get_local_ip)
echo "Active local routing IP: ${ACTIVE_IP}"
```

Yes, HAProxy will multiplex them through a single HTTP/2 TCP connection to ppng.io—but ONLY if you configure your backend reuse policy explicitly. [1] 
If you just write a default HAProxy configuration, it will fail to do this.
Understanding how HAProxy maps incoming requests to a multiplexed HTTP/2 backend clarifies the exact settings required:
## 1. The Concurrent Batch (N Parallel curls)
When you fire a batch of curl commands at the same time:

* 
* Each curl process opens its own independent, plain HTTP/1.1 TCP connection to your local HAProxy frontend.
* HAProxy receives N separate frontend TCP connections simultaneously.
* 

## 2. How HAProxy Maps Them to the Backend (ppng.io)
By default, HAProxy treats backend connections conservatively. To force HAProxy to bundle all those independent frontend requests into one single, shared HTTP/2 connection to ppng.io, you must explicitly configure connection pooling and reuse in your backend. [1, 2] 
You need to add two lines (alpn h2 and http-reuse always) to your config:

backend ppng_backend
    mode http
    # 'http-reuse always' forces HAProxy to aggressively share open connections across different clients
    http-reuse always
    
    # 'alpn h2' tells HAProxy to negotiate HTTP/2 with ppng.io over TLS
    server ppng_server ppng.io:443 ssl verify required ca-file /etc/ssl/certs/ca-certificates.crt alpn h2

## What Happens With This Configuration:

* 
* For the Parallel Batch: The first curl request hitting HAProxy causes it to open a single TCP/TLS connection to ppng.io and negotiate HTTP/2. While that connection is open, the other N-1 parallel curl requests hitting your frontend are instantly assigned their own unique Stream IDs and packed into that exact same backend TCP pipeline. [2, 3] 
* For Sequential Requests: If curl commands are executed one after the other, http-reuse always keeps the backend HTTP/2 connection idling in an "orphan connection pool". When the next sequential curl command hits HAProxy, it reuses that same warm TCP stream, sending the data over a new Stream ID without executing a new TLS handshake. [1, 2, 4] 
* 

## Summary
Without http-reuse always, HAProxy will lazily spin up a new backend TCP connection for every incoming client connection. [5, 6] 
With http-reuse always and alpn h2, HAProxy functions exactly as an upgrade relay: it converts N independent, unencrypted HTTP/1.1 frontend connections into N multiplexed HTTP/2 binary streams over a single, secure TCP pipeline to ppng.io. [1, 2] 

[1] [https://www.haproxy.com](https://www.haproxy.com/blog/http-keep-alive-pipelining-multiplexing-and-connection-pooling)
[2] [https://docs.haproxy.org](https://docs.haproxy.org/2.0/configuration.html)
[3] [https://andreaskaris.github.io](https://andreaskaris.github.io/blog/networking/haproxy-and-h2c/)
[4] [https://discourse.haproxy.org](https://discourse.haproxy.org/t/haproxy-http-reuse-never-option-not-working-for-haproxy-1-9-8-and-2-0-1/4001?page=2)
[5] [https://github.com](https://github.com/haproxy/haproxy/issues/1442)
[6] [https://www.haproxy.com](https://www.haproxy.com/glossary/what-is-connection-reuse)
