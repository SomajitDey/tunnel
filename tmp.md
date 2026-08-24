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
