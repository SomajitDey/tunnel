# Tunnel

Secure, multiplexed, TCP/UDP port forwarder using [piping-server](https://github.com/nwtgck/piping-server) by [@nwtgck](https://github.com/nwtgck) as relay. Designed mainly for p2p connections between peers behind (multiple) NAT/firewalls.

# Features

1. TCP/UDP tunnel between peers, each of which may be behind (multiple) NAT(s), i.e. unreachable from the public internet.
2. Firewalls don't cause problems as only outgoing http(s) connections are used.
3. Security: To connect, peers must know the unique ID of the serving peer and a shared secret key. Traffic between peer and relay is encrypted (TLS). [Relay doesn't store anything](https://github.com/nwtgck/piping-server#ideas).
4. Multiplexing: Each tunnel supports multiple concurrent connections. Connections are full-duplex.
5. Many-to-One: The forwarding peer acts as the client and the forwardee peer acts as the server. Server can support multiple clients at any given time. Each node can act as both server and client.
6. Resilience: Peers auto-reconnect in the face of intermittent connectivity.
7. No superuser privilege required.
8. [Option to host your own relay server (easily and for free)](https://github.com/nwtgck/piping-server#self-host-on-free-services).
9. KISS: Just a single, small, portable, shell-script.
10. Built in installer and updater.
11. Opt-in custom domains: Bring your own domain(s) to replace long, messy peer ID(s).

# Command-line

> [!NOTE]
> For the special case of **IPFS**, see the [examples](#examples) below.

**<u>ID</u>:** Every node is given a unique identifier in [bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki#user-content-Bech32) format -

```bash
tunnel -i
```

Share this ID with your peers once and for all.

**<u>Server mode</u>:** Expose your local port to peers with whom you share any secret string -

```bash
tunnel [options] [-u] [-k <shared-secret>] <local-port>
```

**<u>Client mode</u>:** Forward your local port to peer's exposed local port -

```bash
tunnel [options] [-u] [-k <shared-secret>] [-b <local-port>] [-I <IP>] <peer-ID:peer-port>
```

> [!TIP]
> You may also pass a custom domain instead of the long bech32 `peer-ID` string, as discussed [here](#custom-domains).

If no local-port is provided using the `-b` option, `tunnel` uses a random unused port. The port used, is always reported at stdout.

The `-I` option is handy when client is running on a laptop that occasionally gets connected to the LAN the server is on. When server can be found on LAN with private IP = `<IP>`, `tunnel` connects through LAN.

Client and server must use the same secret to be able to connect with each other. The secret string may also be passed using the environment variable `TUNNEL_KEY`. Secret passed with `-k` takes precedence.

`-u` flag denotes use of UDP instead of the default TCP. If used, it must be used by both the peers.

All logs are at stderr by default. With the `-l <logfile>` option, however, one can launch `tunnel` in background (**<u>daemon mode</u>**) with logs dumped at `<logfile>`. The daemon process ID is shown to the user during launch so that he can kill the daemon anytime with 
```bash
tunnel -K <procID>
```

**<u>Options</u>:** 

​For a full list of options see : `tunnel -h`	

# Installation and Updating

Download with:

```bash
curl -LO https://raw.githubusercontent.com/SomajitDey/tunnel/main/tunnel
```

Make it executable:

```bash
chmod +rx ./tunnel
```

Then install system-wide with:

```bash
./tunnel -c install
```

If you don't have `sudo` privilege, you can install locally instead:

```bash
./tunnel -c install -l
```

To update anytime after installation:

```bash
tunnel -c update
```

# Dependencies

This program is simply an executable `bash` script depending on standard GNU tools, such as `socat`, `openssl`, `curl`, `mktemp`, `cut`, `flock`, `pkill` and `xxd`, that are readily available on standard Linux distros. For providing cryptographic security, it also uses `[age](https://github.com/filosottile/age#installation)`.

If your system lacks any of these tools, and you do not have the `sudo` privilege required to install it from the native package repository (e.g. `sudo apt-get install <package>`), try downloading a [portable binary](https://github.com/ernw/static-toolbox/releases) and install it locally at `${HOME}/.bin`. If nothing works, you can always build and install the required open-source tool locally.

# Examples

**<u>*SSH*</u>:**

Peer A exposes local SSH port -

```bash
tunnel -k "${secret}" 22
```

Peer B connects -

```bash
tunnel -b 67868 -k "${secret}" -l /dev/null "${peerA_ID}:22" # Daemon due to -l
ssh -l "${login_name}" -p 67868 localhost 
```

**<u>*IPFS*</u>:**

Let peer A has [IPFS-peer-ID](https://docs.libp2p.io/concepts/peer-id/): `12orQmAlphanumeric`. Her IPFS daemon listens at default TCP port 4001. She exposes it with -

```bash
tunnel -k "${swarm_key}" ipfs
```

`swarm_key` is just any secret string peer A may use to control who can swarm connect to her using `tunnel`. 

Peer B now connects with peer A for [file-sharing](https://docs.ipfs.io/concepts/usage-ideas-examples/) or [pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-pubsub) or [p2p](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p) -

```bash
tunnel -k "${swarm_key}" 12orQmAlphanumeric
```

This last command swarm connects to peer A through the [piping-server relay](https://ppng.io) and keeps on swarm connecting every few seconds in the background to keep the connection alive. 

`tunnel` starts the IPFS daemon in background if not already active.

The path to IPFS repo may be passed with the option `-r`. Otherwise, the environment variable `IPFS_PATH` or the default path `~/.ipfs` is used as usual. Example: `tunnel -r ~/.ipfs -i` gives the IPFS peer ID.

**<u>*Remote Shell*</u>:**

Suppose you would regularly need to launch commands at your workplace Linux box from your home machine. And you don't want to / can't use SSH over `tunnel` for some reason.

At the workplace computer, expose some random local TCP port, e.g. 49090 and connect a shell to that port:

```bash
tunnel -l "/tmp/tunnel.log" -k "your secret" 49090 # Note the base64 node id emitted
socat TCP-LISTEN:49090,reuseaddr,fork SYSTEM:'bash 2>&1'
```

Back at your  home:

```bash
tunnel -l "/dev/null" -b 5000 -k "your secret" "node_id_of_workplace:49090"
rlwrap nc localhost 5000
```

Using [rlwrap](https://github.com/hanslub42/rlwrap) is not a necessity. But it sure makes the experience sweeter as it uses GNU Readline and remembers the input history (accessible with the up/down arrow keys similar to your local bash sessions).

**<u>*Redis*</u>:**

Need to connect to a remote [Redis](https://redis.io/) instance hosted by a peer or yourself? At the remote host, expose the TCP port that `redis-server` runs on (default: 6379), with `tunnel`.

At your local machine, use `tunnel` to forward a TCP port to the remote port. Point your `redis-cli` at the forwarded local port.

# Custom Domains

If you own a custom domain, you may want to map it to your `tunnel` ID, so that you can pass that domain to the `tunnel` client instead of the long bech32 ID string.

For example, consider you're publicly hosting a web server at `www.example.com:443`. The firewall at the server, however, does not allow incoming connections from the public internet to any port other than `443`. To `ssh` into the server from outside, you'd want to bypass the firewall using `tunnel`. It'd be very convenient if the `tunnel` client could extract the server's long bech32 peer-ID from the hostname (i.e. `www.example.com`) itself.

To achieve this, simply login to your domain registrar or DNS provider and publish your *server's bech32 ID prefixed with `tunnel=`* as a `TXT` record against your *hostname prefixed with `_tunnel.`*.

Once you map your ID to the hostname, say `www.example.com`, verify the following holds:

```bash
$ dig _tunnel.www.example.com TXT +short
# Output of the above command should contain the following line:
tunnel=age12c2950resnl0f5fqjr8r47hnqpnk4qwh3hfs6fvalx6shr7r849qdl2ad5
# Used a random peer-ID for illustration above
```

Now you can launch the `tunnel` client simply as:
```bash
tunnel -k "${secret}" www.example.com:22

# Provided the server's running: tunnel -k "${secret}" 22
```

# Applications

**I dogfooded `tunnel` myself for years to access my University's LAN from my home PC**. Broadly speaking, anything that involves NAT/firewall traversal or accessing a remote node without a public IP, should find `tunnel` useful.

Usually you'd want to expose `tunnel` at the remote node once, yet use multiple services hosted at or available exclusively from there. `sslh` and `SOCKS` are the exact tools for this.

- At the remote node, configure `sslh` to listen to the port that you will expose with `tunnel`. `sslh` sniffs the first bytes of any incoming request and connects it to its desired service as configured.

- A `SOCKS` proxy, on the other hand, can send your request anywhere on the internet. This is the only thing you'd need to access the internet from your peer's (i.e. the remote server) IP address. Setup a `SOCKS` proxy behind the `sslh` at the remote node.

- If WebRTC p2p is needed, setup a TURN server with `coturn` behind `sslh`.

- `tunnel` now represents the remote port (that `sslh` is listening to) as a local port in your PC.

- Point all your client apps to this local port. For web browsing, set up your browser to use the local port as a SOCKS proxy.

I could access paywalled journals subscribed by my University from my home browser for free using this setup. Also used it for remote desktop (as a free alternative to AnyDesk), file transfer and of course SSH. Using the `-I` option meant whenever my local PC and the remote node came on the same LAN (e.g. when I connected my laptop to the office LAN), `tunnel` connected the two directly, bypassing the piping-server relay.

# Security

`tunnel` encrypts all traffic between a peer and the relay with TLS, if the relay uses https. There is no end-to-end encryption *per se* between the peers themselves. However, the piping-server relay is claimed to be *[storageless](https://github.com/nwtgck/piping-server#ideas)*.

A client peer can connect with a serving peer only if they use the same secret key (TUNNEL_KEY). The key is primarily used for peer discovery at the relay stage. For every new connection to the forwarded local port, the client sends a random session key to the serving peer. The peers then form a new connection at another relay point based on this random key for the actual data transfer to occur. Outsiders, viz. bad actors who don't know the TUNNEL_KEY shouldn't be able to guess and disrupt this flow.

~~However, a malicious peer can do the following. Because he knows the TUNNEL_KEY and the node ID of the serving peer, he can impersonate the latter. Data from an unsuspecting connecting peer, therefore, would be forwarded to the impersonator, starving the genuine server. Future updates/implementations of `tunnel` should handle this threat using public key crypto. [In that case, the random session key generated for every new connection to be forwarded, would be decryptable by the genuine server alone].~~ < **This was fixed in v1.0.0**

Currently `tunnel` trusts the piping-server relay as a Man-In-The-Middle (MITM). Most applications that you would use `tunnel` with, such as SSH, HTTPS and IPFS, secure the connection themselves, eliminating any MITM threat. Adding E2EE(TLS) to `tunnel` for *all* data transfers would only add unnecessary overhead. For insecure applications however, you can always create a secure SSH-tunnel after establishing the low-level peering with `tunnel`.

# Relay

The default relay used by `tunnel` is https://ppng.io. You can also use some other public relay from this [list](https://github.com/nwtgck/piping-server#public-servers) or [host your own instance](https://github.com/nwtgck/piping-server#self-host-on-free-services). Needless to say, to connect, two peers must use the same relay.

If you so choose, you can also write your own relay to be used by `tunnel` in your preferred language. Just make sure your relay service has the same API as [piping-server](https://github.com/nwtgck/piping-server). If your relay code is open source, you are most welcome to introduce it at [discussions](https://github.com/SomajitDey/tunnel/discussions).

# See also

[gsocket](https://github.com/hackerschoice/gsocket) ; [ipfs p2p](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p) with [circuit-relay enabled](https://gist.github.com/SomajitDey/7c17998825bb105466ef2f9cefdc6d43) ; [go-piping-duplex](https://github.com/nwtgck/go-piping-duplex) ; [pipeto.me](https://pipeto.me) ; [uplink](https://getuplink.de) ; [localhost.run](https://localhost.run/) ; [ngrok](https://ngrok.io) ; [localtunnel](https://github.com/localtunnel/localtunnel) ; [sshreach.me](https://sshreach.me) (*free trial for limited period only*) ; [httprelay.io](https://httprelay.io) ; [more](https://gist.github.com/SomajitDey/efd8f449a349bcd918c120f37e67ac00)

**Notes:** 

1. Unlike [piping-server](https://github.com/nwtgck/piping-server), most of these do not offer [easy, free self-hosting](https://github.com/nwtgck/piping-server#self-host-on-free-services) of the all-important relay or reverse proxy. If these services ever go down, you are doomed. With `tunnel` and [piping-server](https://github.com/nwtgck/piping-server), however, you can simply deploy your own relay instance, share its public URL with your peers once and for all, `export` the same as `TUNNEL_RELAY` inside `.bashrc` and you are good to go. Also, multiple [public piping-servers](https://github.com/nwtgck/piping-server#public-servers) are available for redundancy.
2. Some of these services, in the free tier, give a new random public URL for every session, which is problematic for intermittent connectivity (in IOT applications for example). Some free plans also expire sessions after a certain time even if connections are not idling.
3. Some expose your local port for web-traffic only. The onus of transporting non-web protocols over HTTP is left on you and your peers.

# Future directions

**IPFS (Done):** 

Connecting to IPFS would be much simpler: 

`tunnel -k <secret> ipfs` to expose and `tunnel -k <secret> <IPFS_peerID>` to connect. 

These will launch the IPFS daemon on their own, if offline. The latter command will repeatedly swarm connect to the given peer at 30s intervals. The IPFS-peer-ID will be used as the node ID, so peers would no more need to share their node IDs separately. Non-default IPFS repo paths may be passed with option `-r`. or `IPFS_PATH`.

**SSH:**

Creating an SSH tunnel between local and peer port would be as easy as:

`tunnel -k <secret> ssh` to expose &

`tunnel -sk <secret> -b <local-port> <peerID>:<peer-port>` to create.

Note that, while connecting, one no more needs to provide a login name. The `${USER}` of the serving node is taken as the login name by default. However, if needed, a non-default login name can always be passed using an environment variable or option.

~~**GPG:**~~

~~Virtual machines, such as used by cloud-shells and dynos, do not have persistent, unique hardware addresses. The node ID therefore keeps on changing from session to session for such a VM. Future `tunnel` would have a `-g` option which would pass a GPG private key to  `tunnel`. The node ID would be generated from the fingerprint of this key, [akin to what IPFS does](https://docs.libp2p.io/concepts/peer-id/). This would also make `tunnel` more secure.~~ < This was implemented in v1.0.0 with `age` instead of GPG.

**Argon2:**

Option [`-a`] to use [argon2](https://github.com/P-H-C/phc-winner-argon2) for hashing TUNNEL_KEY before use, so that a weaker secret isn't too vulnerable.

**Optional E2EE (TLS):**

Implementable using `stunnel` with `PSKsecrets`. This feature must be optional as most apps secure the connection themselves.

# Contribute

- Please report bugs at [issues](https://github.com/SomajitDey/tunnel/issues). 

- Post your thoughts, comments, ideas, use-cases and feature-requests at [discussion](https://github.com/SomajitDey/tunnel/discussions).

- If you're using this little tool in your daily life, please :star: this repository ... and share it with others!

- If you're feeling generous today, click the **Sponsor** button on this page :smiley:

------

###### [Copyright](https://github.com/SomajitDey/tunnel/blob/main/LICENSE) &copy; 2021-2026 [Somajit Dey](https://github.com/SomajitDey)
