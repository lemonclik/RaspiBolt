---
layout: default
title: Lightning
nav_order: 5
---
# Lightning: LND
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

Now we will set up LND, the Lightning Network Daemon by [Lightning Labs](http://lightning.engineering/). Check out their [Github repository](https://github.com/lightningnetwork/lnd/blob/master/README.md) for a wealth of information about their open-source project and Lightning in general.

## Dowload LND
Download and install LND
```
$ cd /home/admin/download
$ rm -rf /home/admin/download/*
$ wget https://github.com/lightningnetwork/lnd/releases/download/v0.7.1-beta/lnd-linux-armv7-v0.7.1-beta.tar.gz
$ wget https://github.com/lightningnetwork/lnd/releases/download/v0.7.1-beta/manifest-v0.7.1-beta.txt
$ wget https://github.com/lightningnetwork/lnd/releases/download/v0.7.1-beta/manifest-v0.7.1-beta.txt.sig
$ wget https://keybase.io/roasbeef/pgp_keys.asc

$ sha256sum --check manifest-v0.7.1-beta.txt --ignore-missing
> lnd-linux-armv7-v0.7.1-beta.tar.gz: OK

$ gpg ./pgp_keys.asc
> BD599672C804AF2770869A048B80CD2BB8BD8132

$ gpg --import ./pgp_keys.asc
$ gpg --verify manifest-v0.7.1-beta.txt.sig
> gpg: Good signature from "Olaoluwa Osuntokun <laolu32@gmail.com>" [unknown]
> Primary key fingerprint: BD59 9672 C804 AF27 7086  9A04 8B80 CD2B B8BD 8132
>      Subkey fingerprint: F803 7E70 C12C 7A26 3C03  2508 CE58 F7F8 E20F D9A2

$ tar -xzf lnd-linux-armv7-v0.7.1-beta.tar.gz
$ sudo install -m 0755 -o root -g root -t /usr/local/bin lnd-linux-armv7-v0.7.1-beta/*
$ lnd --version
> lnd version 0.7.1-beta commit=v0.7.1-beta
```
![Checksum LND](images/40_checksum_lnd.png)

## LND configuration
Now that LND is installed, we need to configure it to work with Bitcoin Core and run automatically on startup.

* Still as user 'admin', create a symbolic link to the `ip` binary located in `/bin/ip`, as it seems as LND cannot find it in some cases.  
  `$ sudo ln -s /bin/ip /usr/bin/ip`

* Open a "bitcoin" user session  
  `$ sudo su - bitcoin` 

* Create the LND working directory and the corresponding symbolic link  
  `$ mkdir /mnt/hdd/lnd`  
  `$ ln -s /mnt/hdd/lnd /home/bitcoin/.lnd`  
  `$ ls -la`

![Check symlink LND](images/40_symlink_lnd.png)

* Create the LND configuration file and paste the following content (adjust to your alias). Save and exit.  
  `$ nano /home/bitcoin/.lnd/lnd.conf`

```bash
# RaspiBolt: lnd configuration
# /home/bitcoin/.lnd/lnd.conf

[Application Options]
debuglevel=info
maxpendingchannels=5
alias=YOUR_NAME [LND]
color=#68F442

# Your router must support and enable UPnP, otherwise delete this line  
nat=true

[Bitcoin]
bitcoin.active=1

# enable either testnet or mainnet
bitcoin.testnet=1
#bitcoin.mainnet=1

bitcoin.node=bitcoind

[autopilot]
autopilot.active=1
autopilot.maxchannels=5
autopilot.allocation=0.6
```
Some explanations about this configuration:

* The configuration option **nat=true** expects your internet router to support Universal Plug'n'Play (UPnP) and have it enabled. This allows LND to make your node reachable from outside your network by setting up port forwarding, announce your external ip address and update this information if your ip address changes. This is currently the only reliable configuration to have a routing Lightning node. 

  **If your router does not support UPnP**, LND will still work, but your node will be a private Lightning node for your own payments and not able to route payments for others. In this case, you need to delete the `nat=true` line in the configuration file above, otherwise LND will not start. Another option is to pass your ip address on LND start (see [external guide](https://github.com/robclark56/RaspiBolt-Extras/blob/master/RB_extra_02.md)).

* The configuration above has **autopilot** enabled and will automatically open up to **5** channels with **60%** of your funds. You might want to adjust this to suit your needs. With `autopilot.active=0` you can disable the autopilot completely and manage channels manually.

:point_right: Additional information: [sample-lnd.conf](https://github.com/lightningnetwork/lnd/blob/master/sample-lnd.conf) in the LND project repository

## Run LND

Again, we switch to the user "bitcoin" and first start the program manually to check if everything works fine.

```
$ sudo su - bitcoin
$ lnd
```

The daemon prints the status information directly to the command line. This means that we cannot use that session without stopping the server. We need to open a second SSH session.

## LND wallet setup

Start your SSH program (eg. PuTTY) a second time, connect to the Pi and log in as "admin". Commands for the **second session** start with the prompt `$2` (which must not be entered).

Once LND is started, the process waits for us to create the integrated Bitcoin wallet (it does not use the "bitcoind" wallet). 

* Start a "bitcoin" user session   
  `$2 sudo su - bitcoin`

* Create the LND wallet  

  `$2 lncli --network=testnet create` 

* If you want to create a new wallet, enter your `password [C]` as wallet password, select `n` regarding an existing seed and enter the optional `password [D]` as seed passphrase. A new cipher seed consisting of 24 words is created.

![LND new cipher seed](images/40_cipher_seed.png)

These 24 words, combined with your passphrase (optional `password [D]`)  is all that you need to restore your Bitcoin wallet and all Lighting channels. The current state of your channels, however, cannot be recreated from this seed, this requires a continuous backup and is still under development for LND.

:warning: This information must be kept secret at all times. **Write these 24 words down manually on a piece of paper and store it in a safe place.** This piece of paper is all an attacker needs to completely empty your wallet! Do not store it on a computer. Do not take a picture with your mobile phone. **This information should never be stored anywhere in digital form.**

* exit "bitcoin" user session  
  `$2 exit`

Let's authorize the "admin" user to work with LND using the command line interface `lncli`. For that to work, we need to copy the Transport Layer Security (TLS) certificate and the permission files (macaroons) to the admin home folder.

* Check if the TLS certificates have been created  
  `$2 sudo ls -la /home/bitcoin/.lnd/`

* Check if permission files `admin.macaroon` and `readonly.macaroon` have been created.  
  `$2 sudo ls -la /home/bitcoin/.lnd/data/chain/bitcoin/testnet/`

![Check macaroon](images/40_ls_macaroon.png)

* Copy permission files and TLS cert to user "admin"   
  `$2 cd /home/bitcoin/`  
  `$2 sudo cp --parents .lnd/data/chain/bitcoin/testnet/admin.macaroon /home/admin/`  
  `$2 sudo cp /home/bitcoin/.lnd/tls.cert /home/admin/.lnd`  
  `$2 sudo chown -R admin:admin /home/admin/.lnd/`  
* Make sure that `lncli` works by unlocking your wallet (enter `password [C]` ) and getting some node infos.   
  `$2 lncli --network=testnet unlock`
* Check the current state of LND 
  `$2 lncli --network=testnet getinfo`

You can also see the progress of the initial sync of LND with Bitcoin in the first SSH session. 

Check for the following two lines to make sure that the port forwarding is successfully set up using UPnP. If LND is not able to configure your router (that may not support UPnP, for example), your node will still work, but it will not be able to router transactions for other network participants.

```
[INF] SRVR: Scanning local network for a UPnP enabled device
[INF] SRVR: Automatically set up port forwarding using UPnP to advertise external IP
```

Let's stop the server for the moment and focus on our primary SSH session again.

* `$2 lncli --network=testnet stop`
* `$2 exit`

This should terminate LND "gracefully" in SSH session 1 that can now be used interactively again.

## Autostart LND

Now, let's set up LND to start automatically on system startup.

* Exit the "bitcoin" user session back to "admin"  
  `$ exit`
* Create LND systemd unit and with the following content. Save and exit.  
  `$ sudo nano /etc/systemd/system/lnd.service` 

```bash
# RaspiBolt: systemd unit for lnd
# /etc/systemd/system/lnd.service

[Unit]
Description=LND Lightning Daemon
Wants=bitcoind.service
After=bitcoind.service

[Service]
ExecStart=/usr/local/bin/lnd

User=bitcoin
Group=bitcoin
Type=simple
KillMode=process
LimitNOFILE=128000
TimeoutSec=240
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

* Enable, start and unlock LND  
  `$ sudo systemctl enable lnd`  
  `$ sudo systemctl start lnd`  
  `$ systemctl status lnd`  
  `$ lncli --network=testnet unlock`

* Now, the daemon information is no longer displayed on the command line but written into the system journal. You can monitor the LND startup progress until it caught up with the testnet blockchain (about 1.3m blocks at the moment). This can take up to 2 hours, after that you see a lot of very fast chatter (exit with `Ctrl-C`).  
  `$ sudo journalctl -f -u lnd`

![LND startup log](images/40_start_lnd.png)



## Get some testnet Bitcoin

Now your Lightning node is ready. To use it in testnet, you can get some free testnet bitcoin from a faucet.
* Generate a new Bitcoin address to receive funds on-chain  
  `$ lncli --network=testnet newaddress np2wkh`  
  `> "address": "2NCoq9q7............dkuca5LzPXnJ9NQ"` 

* Get testnet bitcoin:  
  https://testnet-faucet.mempool.co

* Check your LND wallet balance  
  `$ lncli --network=testnet walletbalance`  

* Monitor your transaction (the faucet shows the TX ID) on a Blockchain explorer:  
  https://testnet.smartbit.com.au

### LND in action
As soon as your funding transaction is mined and confirmed, LND will start to open and maintain channels. This feature is called "Autopilot" and is configured in the "lnd.conf" file. If you would like to maintain your channels manually, you can disable the autopilot.

Get yourself a payment request on [StarBlocks](https://starblocks.acinq.co/#/) or [Y’alls](https://testnet.yalls.org/) and move some coins!

* `$ lncli --network=testnet listpeers`  
* `$ lncli --network=testnet listchannels`  
* `$ lncli --network=testnet sendpayment --pay_req=lntb32u1pdg7p...y0gtw6qtq0gcpk50kww`  
* `$ lncli --network=testnet listpayments`  

:point_right: see [Lightning API reference](http://api.lightning.community/) for additional information

----

## Additional topics
### (Optional) Add aliases for easier commands
If you don't want to type out the full commands each time, aliases will help. See the [Additional scripts](raspibolt_67_additional-scripts.md) section for alias setup.

### LND upgrade
If you want to upgrade to a new release of LND in the future, check out the FAQ section:  
[How to upgrade LND](raspibolt_faq.md#how-to-upgrade-lnd)

-----

## Before proceeding to mainnet 
This is the point of no return. Up until now, you can just start over. Experiment with testnet bitcoin. Open and close channels on the testnet. 

Once you switch to mainnet and send real bitcoin to your RaspiBolt, you have "skin in the game". 

* Make sure your RaspiBolt is working as expected.
* Get a little practice with `bitcoin-cli` and its options (see [Bitcoin Core RPC documentation](https://bitcoin-rpc.github.io/))
* Do a dry run with `lncli` and its many options (see [Lightning API reference](http://api.lightning.community/))
* Try a few restarts (`sudo shutdown -r now`), is everything starting fine?

---
Next: [Mainnet >>](raspibolt_50_mainnet.md)
