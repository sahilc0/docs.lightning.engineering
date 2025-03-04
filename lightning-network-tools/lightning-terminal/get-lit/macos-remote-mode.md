# MacOS - Remote Mode

## Assumptions

1. You have a Ubuntu server **already running** `lnd`.
2. Your Ubuntu server has `bitcoind` installed, synced on mainnet, and running as a daemon.
3. 1. To install `bitcoind` on Ubuntu, follow [the official bitcoin.org guide](https://bitcoin.org/en/full-node#linux-instructions).
4. Your `bitcoin.conf` file has ZMQ publishing turned on.
   1. Txindex is not required, but pruned mode is not supported.

## Get LiT

Download the version of the latest [Lightning Terminal release](https://github.com/lightninglabs/lightning-terminal/releases/latest) that matches your local configuration \(likely darwin-amd64\). Extract the compressed files, and install the binaries using the below instructions.

```text
# Extract the compressed files, and install them in your GOPATH
# GOPATH by default should be /usr/local/go/bin
tar -xvf lightning-terminal-darwin-amd64-<YOUR_LOCAL_VERSION>.tar.gz --strip 1 -C $GOPATH/bin

# If GOPATH is not defined, add your current folder to your PATH for simplicity
tar -xvf lightning-terminal-darwin-amd64-<YOUR_LOCAL_VERSION>.tar.gz
cd lightning-terminal-darwin-amd64-<YOUR_LOCAL_VERSION>
PATH=$PATH:$PWD
```

Ensure that your Ubuntu server has only the required ports open for outbound communication with the Lightning Network.

```text
sudo ufw logging on
sudo ufw enable
# PRESS Y
sudo ufw status
sudo ufw allow OpenSSH
sudo ufw allow 9735
sudo ufw allow 10009
sudo ufw allow 443
```

To connect Lightning Terminal to a remote `lnd` instance first make sure your remote `lnd.conf` file contains the following additional configuration settings:

```text
tlsextraip=<YOUR_LND_IP>
rpclisten=0.0.0.0:10009
```

Copy the following files that are located in your `~/.lnd/data/chain/bitcoin/mainnet` directory on your remote machine to `/some/folder/with/lnd/data/` on your local machine \(where you’ll be running LiT\):

* tls.cert
* admin.macaroon

Create a `lit.conf` file.

```text
mkdir ~/Library/Application\ Support/Lit/
vi ~/Library/Application\ Support/Lit/lit.conf
```

Storing the configuration in a persistent `~/Library/Application Support/Lit/lit.conf` file means you do not need to type in the command line arguments every time you start the server. Make sure you don't add any section headers \(the lines starting with \[ and ending with \], for example \[Application Options\]\) as these don't work with the additional levels of sub configurations.

Paste this example `lit.conf` file into your terminal, and fill in the placeholders with your specific information.

```text
# Application Options: lnd-mode not required since remote is default
httpslisten=0.0.0.0:443
lit-dir=~/Library/Application Support/Lit/

# Let's Encrypt
# You can configure the HTTPS server to automatically install a free SSL certificate provided by Let's Encrypt. 
# This is recommended if you plan to access the website from a remote computer, but does require extra setup.
#letsencrypt=true
#letsencrypthost=<YOUR_DOMAIN>

# Network setting
network=testnet

# Remote options
remote.lit-debuglevel=debug

# Remote lnd options
remote.lnd.rpcserver=<YOUR_LND_IP>:10009
remote.lnd.macaroonpath=/path/to/remote/files/you/copied/admin.macaroon
remote.lnd.tlscertpath=/path/to/remote/files/you/copied/tls.cert

# Loop
loop.loopoutmaxparts=5

# Pool
pool.newnodesonly=true

# Faraday
faraday.min_monitored=48h

# Faraday - bitcoin
faraday.connect_bitcoin=true
faraday.bitcoin.host=localhost
faraday.bitcoin.user=<YOUR_RPCUSER>
faraday.bitcoin.password=<YOUR_RPCPASSWORD>
```

If you are using a cloud provider, double check using their configuration tools that inbound ports 443, 9735, and 10009 are allowed. Once you've done that, and you've ensured your remote `lnd` instance is running, it's time to get LiT!

```text
litd --uipassword=<YOUR_UI_PASSWORD>
```

Visit [https://localhost:8443](https://localhost:8443/) to access LiT.

## Example commands for interacting with the command line

Because not all functionality of `lnd` \(or `loop`/`faraday` for that matter\) is available through the web UI, it will still be necessary to interact with those daemons through the command line.

We are going through an example for each of the command line tools and will explain the reasons for the extra flags. The examples assume that LiT is started with the following configuration \(only relevant parts shown here\):

```text
lnd-mode=integrated

lnd.lnddir=~/Library/Application Support/Lnd
lnd.rpclisten=0.0.0.0:10009

lnd.bitcoin.testnet=true
```

Because all components listen on the same gRPC port and use the same TLS certificate, some command line calls now need some extra options that weren't necessary before.

**NOTE**: All mentioned command line tools have the following behavior in common: You either specify the `--network` flag and the `--tlscertpath` and `--macaroonpath` are implied by looking inside the default directories for that network. Or you specify the `--tlscertpath` and `--macaroonpath` flags explicitly, then you **must not** set the `--network` flag. Otherwise, you will get an error like: `[lncli] could not load global options: unable to read macaroon path (check the network setting!): open /Users/<user>/Library/Application Support/Lnd/data/chain/bitcoin/testnet/admin.macaroon: no such file or directory`

### Example `lncli` command

The `lncli` commands in the "integrated" mode are the same as if `lnd` was running standalone. The `--lnddir` flag does not need to be specified as long as it is the default directory \(`~/.lnd` on Linux\).

```text
$ lncli --network=testnet getinfo
```

### Example `loop` command

This is where things get a bit tricky. Because as mentioned above, `loopd` also runs on the same gRPC server as `lnd`. That's why we have to both specify the `host:port` as well as the TLS certificate of `lnd`. But `loopd` verifies its own macaroon, so we have to specify that one from the `.loop` directory.

```text
$ loop --rpcserver=localhost:10009 --tlscertpath=~/Library/Application\ Support/Lnd/tls.cert \
  --macaroonpath=~/Library/Application\ Support/Loop/testnet/loop.macaroon \
  quote out 500000
```

You can easily create an alias for this by adding the following line to your `~/.bashrc` file:

```text
alias lit-loop="loop --rpcserver=localhost:10009 --tlscertpath=~/Library/Application\ Support/Lnd/tls.cert --macaroonpath=~/Library/Application\ Support/Loop/testnet/loop.macaroon"
```

### Example `pool` command

Again, `poold` also runs on the same gRPC server as `lnd` and we have to specify the `host:port` and the TLS certificate of `lnd` but use the macaroon from the `.pool` directory.

```text
$ pool --rpcserver=localhost:10009 --tlscertpath=~/Library/Application\ Support/Lnd/tls.cert \
  --macaroonpath=~/Library/Application\ Support/Pool/testnet/pool.macaroon \
  accounts list
```

You can easily create an alias for this by adding the following line to your `~/.bashrc` file:

```text
alias lit-pool="pool --rpcserver=localhost:10009 --tlscertpath=~/Library/Application\ Support/Lnd/tls.cert --macaroonpath=~/Library/Application\ Support/Pool/testnet/pool.macaroon"
```

### Example `frcli` command

Faraday's command line tool follows the same pattern as loop. We also have to specify the server and TLS flags for `lnd` but use `faraday`'s macaroon:

```text
$ frcli --rpcserver=localhost:10009 --tlscertpath=~/Library/Application\ Support/Lnd/tls.cert \
  --macaroonpath=~/Library/Application\ Support/Faraday/testnet/faraday.macaroon \
  audit
```

You can easily create an alias for this by adding the following line to your `~/.bashrc` file:

```text
alias lit-frcli="frcli --rpcserver=localhost:10009 --tlscertpath=~/Library/Application\ Support/Lnd/tl
```

