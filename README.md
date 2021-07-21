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

# Command-line

For the special case of **IPFS**, see [#examples](#examples) below.

**<u>ID</u>:** Every node is given a unique ([base64](https://datatracker.ietf.org/doc/html/rfc2045#page-24)) ID -

```bash
tunnel -i
```

ID is bound to hardware (MAC address) and the environment variables USER, HOME and HOSTNAME. Share it with your peers once and for all. Note: two users on the same machine are given separate node-IDs because their USER and HOME variables differ.

**<u>Server mode</u>:** Expose your local port to peers with whom you share any secret string -

```bash
tunnel [options] [-u] [-k <shared-secret>] <local-port>
```

**<u>Client mode</u>:** Forward your local port to peer's exposed local port -

```bash
tunnel [options] [-u] [-k <shared-secret>] [-b <local-port>] <peer-ID:peer-port>
```

If no local-port is provided using the `-b` option, `tunnel` uses a random unused port. The port used, is always reported at stdout.

Client and server must use the same secret to be able to connect with each other. The secret string may also be passed using the environment variable `TUNNEL_KEY`. Secret passed with `[-k]` takes precedence.

`[-u]` flag denotes use of UDP instead of the default TCP. If used, it must be used by both the peers.

All logs are at stderr by default.

**<u>Options</u>:** 

​	**-v**  Version

​	**-h**  Help

​	**-p**  *"\<piping-server URL\>*"

​		Can use environment variable `TUNNEL_RELAY` instead. Default: https://ppng.io. See [list](https://github.com/nwtgck/piping-server#public-servers).

​	**-l**   *"\<path to log file\>*"

​		Runs `tunnel` as daemon with stderr redirected to given path.

# Installation and Updating

Download with:

```bash
curl -LO https://raw.githubusercontent.com/SomajitDey/tunnel/main/tunnel
```

Make it executable:

```bash
chmod +x ./tunnel
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

# Dependency/Portability

This program is simply an executable `bash` script depending on standard GNU tools including `socat`, `openssl`, `curl`, `mktemp`, `cut`, `awk`,  `sed` , `flock`, `pkill`, `dd`, `xxd`, `base64` etc. that are readily available on standard Linux distros.

If your system lacks any of these tools, and you do not have the `sudo` privilege required to install it from the native package repository (e.g. `sudo apt-get install <package>`), try downloading a [portable binary](https://github.com/ernw/static-toolbox/releases) and install it locally at `${HOME}/.bin`.

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

# Applications

Below are some random use-cases I could think of for `tunnel`. Broadly speaking, anything that involves NAT/firewall traversal (e.g. WebRTC without TURN) or joining a remote LAN, should find `tunnel` useful. Some of the following ideas are rather sketchy, haven't been tested at all, and may not work, but nonetheless are documented here, at least for the time being, just for the sake of inspiration. If you found any of these useful, or useless, or you have found entirely new applications for `tunnel`, please post at [discussions](https://github.com/SomajitDey/tunnel/discussions). Those cases that I have tested are labelled as "working".

- Connecting to [IPFS](https://docs.ipfs.io/concepts/what-is-ipfs/#decentralization) peers (*Working*).
- P2P chatting, mailing, VoIP, streaming, gaming, screen-sharing, file-sharing, gambling, troubleshooting and what not.
- Connecting IOT devices.
- Connecting your workstation with your home-computer or laptop with SSH (*Working*), [RDP](https://en.wikipedia.org/wiki/Remote_Desktop_Protocol) or [VNC](https://en.wikipedia.org/wiki/Virtual_Network_Computing).
- [Shell-shovelling](https://en.wikipedia.org/wiki/Shell_shoveling) (*Working*).
- Beat your firewall with a self-hosted or peer-provided VPN.
- Joining intranet office chats (over office LAN) from your home across the internet. For example, [BeeBEEP](https://www.beebeep.net/) and [LAN Messenger](https://lanmessenger.github.io/) may use a local port that has been forwarded to from a node inside your office using `tunnel`.
- P2P (serverless) audio/video. For example, forward a local TCP/UDP port to peer's localhost and point your p2p client to it.  Ready-made FOSS: [Jami](https://jami.net/), [Jitsi](https://jitsi.org/), [Toxchat](https://tox.chat/) and [Retroshare](https://retroshare.cc/).
- Serverless remote-control using [Teamviewer](https://community.teamviewer.com/English/kb/articles/4618-can-teamviewer-be-used-within-a-local-network-lan-only) or [Anydesk](https://support.anydesk.com/Settings#Direct_Connection). Forward a local port, say 6000, to peer's local TCP port [5938](https://community.teamviewer.com/English/kb/articles/4139-which-ports-are-used-by-teamviewer) (for TV) and [7070](https://support.anydesk.com/Settings#Local_Port_Listening) (for Anydesk) and use *127.0.0.1:6000* as the *IP:port* tuple of the peer.
- Making a local (web) port publicly accessible, *a.k.a* reverse-proxy accessible from the internet: Simply run `tunnel` at Heroku (for free) and forward the port stored in the environment variable `PORT` to your local port that you want to expose. And you have your public URL as: https://your-app-name.herokuapp.com.
- Accessing feed from a remote [IP webcam](https://play.google.com/store/apps/details?id=com.pas.webcam&hl=en_IN&gl=US) using a [universal network camera adapter](http://ip-webcam.appspot.com/): Forward a local port to a remote node that can access the mobile cam over [WLAN](https://en.wikipedia.org/wiki/Wireless_LAN).
- Streaming with VLC: [Movies and Music](https://www.howtogeek.com/118075/how-to-stream-videos-and-music-over-the-network-using-vlc/), [Webcam](https://forums.tomsguide.com/faq/how-to-stream-videos-over-the-internet-with-vlc.23235/) [[command-line](https://medium.com/@petehouston/streaming-webcam-to-http-using-vlc-dda7259176c9)]. Just connect the server and client using `tunnel`.

# Security

`tunnel` encrypts all traffic between a peer and the relay with TLS, if the relay uses https. There is no end-to-end encryption *per se* between the peers themselves. However, the piping-server relay is claimed to be *[storageless](https://github.com/nwtgck/piping-server#ideas)*.

A client peer can connect with a serving peer only if they use the same secret key (TUNNEL_KEY). The key is primarily used for peer discovery at the relay stage. For every new connection to the forwarded local port, the client sends a random session key to the serving peer. The peers then form a new connection at another relay point based on this random key for the actual data transfer to occur. Outsiders, viz. bad actors who don't know the TUNNEL_KEY shouldn't be able to disrupt this flow.

However, a malicious peer can do the following. Because he knows the TUNNEL_KEY and the node ID of the serving peer, he can impersonate the latter. Data from an unsuspecting connecting peer, therefore, would be forwarded to the impersonator, starving the genuine server. Future updates/implementations of `tunnel` should handle this threat using public key crypto. [In that case, the random session key generated for every new connection to be forwarded, would be decryptable by the genuine server alone].

Given that `tunnel` is essentially the transport layer, the above points should not be discouraging, because most applications such as SSH and IPFS encrypt data at the application layer. Encrypting `tunnel` end-to-end for *all* data transfers would only add to the latency. However, you can always create an SSH-tunnel after establishing the low-level peering with `tunnel`, if you so choose.

# Relay

The default relay used by `tunnel` is https://ppng.io. You can also use some other public relay from this [list](https://github.com/nwtgck/piping-server#public-servers) or [host your own instance](https://github.com/nwtgck/piping-server#self-host-on-free-services) on free services such as offered by [Heroku](https://www.heroku.com/). Needless to say, to connect, two peers must use the same relay.

If you so choose, you can also write your own relay to be used by `tunnel` using simple tools like [sertain](https://github.com/SomajitDey/sertain). Just make sure your relay service has the same API as [piping-server](https://github.com/nwtgck/piping-server). If your relay code is open source, you are most welcome to introduce it at [discussions](https://github.com/SomajitDey/tunnel/discussions).

# See also

[gsocket](https://github.com/hackerschoice/gsocket) ; [ipfs p2p](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p) with [circuit-relay enabled](https://gist.github.com/SomajitDey/7c17998825bb105466ef2f9cefdc6d43) ; [go-piping-duplex](https://github.com/nwtgck/go-piping-duplex) ; [pipeto.me](https://pipeto.me) ; [uplink](https://getuplink.de) ; [localhost.run](https://localhost.run/) ; [ngrok](https://ngrok.io) ; [localtunnel](https://github.com/localtunnel/localtunnel) ; [sshreach.me](https://sshreach.me) (*free trial for limited period only*) ; [more](https://gist.github.com/SomajitDey/efd8f449a349bcd918c120f37e67ac00)

**Notes:** 

1. Unlike [piping-server](https://github.com/nwtgck/piping-server), most of these do not offer [easy, free self-hosting](https://github.com/nwtgck/piping-server#self-host-on-free-services) of the all-important relay or reverse proxy. If these services ever go down, you are doomed. With `tunnel` and [piping-server](https://github.com/nwtgck/piping-server), however, you can simply deploy your own relay instance, share its public URL with your peers once and for all, `export` the same as `TUNNEL_RELAY` inside `.bashrc` and you are good to go. Also, multiple [public piping-servers](https://github.com/nwtgck/piping-server#public-servers) are available for redundancy.
2. Some of these services, in the free tier, give a new random public URL for every session, which is problematic for intermittent connectivity (in IOT applications for example). Some free plans also expire sessions after a certain time even if connections are not idling.
3. Some expose your local port for web-traffic only. The onus of transporting non-web protocols over HTTP is left on you and your peers.

# Future directions

**IPFS:** 

Connecting to IPFS would be much simpler: 

`tunnel -k <secret> ipfs` to expose and `tunnel -k <secret> <IPFS_peerID>` to connect. 

These will launch the IPFS daemon on their own, if offline. The latter command will repeatedly swarm connect to the given peer at 30s intervals. The IPFS-peer-ID will be used as the node ID, so peers would no more need to share their node IDs separately. Non-default IPFS repo paths may be passed with option `-r`. or `IPFS_PATH`.

**SSH:**

Creating an SSH tunnel between local and peer port would be as easy as:

`tunnel -k <secret> ssh` to expose &

`tunnel -sk <secret> -b <local-port> <peerID>:<peer-port>` to create.

Note that, while connecting, one no more needs to provide a login name. The `${USER}` of the serving node is taken as the login name by default. However, if needed, a non-default login name can always be passed using an environment variable or option.

**GPG:**

Virtual machines, such as used by cloud-shells and dynos, do not have persistent, unique hardware addresses. The node ID therefore keeps on changing from session to session for such a VM. Future `tunnel` would have a `-g` option which would pass a GPG private key to  `tunnel`. The node ID would be generated from the fingerprint of this key, [akin to what IPFS does](https://docs.libp2p.io/concepts/peer-id/). This would also make `tunnel` more secure.

**Argon2:**

Option [`-a`] to use [argon2](https://github.com/P-H-C/phc-winner-argon2) for hashing TUNNEL_KEY before use, so that a weaker secret isn't too vulnerable.

# Bug-reports and Feedbacks

Please report bugs at [issues](https://github.com/SomajitDey/tunnel/issues). Post your thoughts, comments, ideas, use-cases and feature-requests at [discussion](https://github.com/SomajitDey/tunnel/discussions). Let me know how this helped you, if it did at all.

Also feel free to write to me [directly](mailto://hereistitan@gmail.com) about anything regarding this project.

If [this](https://github.com/SomajitDey/tunnel/blob/main/tunnel) little script is of any use to you, a [star](https://github.com/SomajitDey/tunnel/stargazers) would be immensely encouraging for me. 

Thanks ! :smiley:

------

###### [Copyright](https://github.com/SomajitDey/tunnel/blob/main/LICENSE) &copy; 2021 [Somajit Dey](https://github.com/SomajitDey)

