[ [Intro](README.md) ] -- [ [Preparations](raspibolt_10_preparations.md) ] -- [ [Raspberry Pi](raspibolt_20_pi.md) ] -- [ [Bitcoin](raspibolt_30_bitcoin.md) ] -- [ **Lightning** ] -- [ [Mainnet](raspibolt_50_mainnet.md) ] -- [ [FAQ](raspibolt_faq.md) ]

-------
### Beginner’s Guide to ️⚡Lightning️⚡ on a Raspberry Pi
--------

# Lightning: LND

We will download and install the LND (Lightning Network Daemon) by [Lightning Labs](http://lightning.engineering/). Check out their [Github repository](https://github.com/lightningnetwork/lnd/blob/master/README.md) for a wealth of information about their open-source project and Lightning in general.

### Public IP script
To announce our public IP address to the Lightning network, we first need to get our address from a source outside our network. As user “admin”, create the following script that checks the IP every 10 minutes and stores it locally.

* Create the following script:  
  `$ sudo nano /usr/local/bin/getpublicip.sh`
```bash
#!/bin/bash
# RaspiBolt LND Mainnet: script to get public ip address
# /usr/local/bin/getpublicip.sh

echo 'getpublicip.sh started, writing public IP address every 10 minutes into /run/publicip'
while [ 0 ]; 
    do 
    printf "PUBLICIP=$(curl -vv ipinfo.io/ip 2> /run/publicip.log)\n" > /run/publicip;
    sleep 600
done;
```
* make it executable  
  `$ sudo chmod +x /usr/local/bin/getpublicip.sh`

* create corresponding systemd unit, save and exit  
  `$ sudo nano /etc/systemd/system/getpublicip.service`

```bash
# RaspiBolt LND Mainnet: systemd unit for getpublicip.sh script
# /etc/systemd/system/getpublicip.service

[Unit]
Description=getpublicip.sh: get public ip address from ipinfo.io
After=network.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/getpublicip.sh
Restart=always

RestartSec=600
TimeoutSec=10

[Install]
WantedBy=multi-user.target
```
* enable systemd startup  
  `$ sudo systemctl enable getpublicip`  
  `$ sudo systemctl start getpublicip`  
  `$ sudo systemctl status getpublicip`  

* check if data file has been created  
  `$ cat /run/publicip`
  `PUBLICIP=91.190.22.151`

### Install LND
Now to the good stuff: download, verify and install the LND binaries.
```
$ wget https://github.com/lightningnetwork/lnd/releases/download/v0.4-beta/lnd-linux-arm-v0.4-beta.tar.gz
$ wget https://github.com/lightningnetwork/lnd/releases/download/v0.4-beta/manifest-v0.4-beta.txt
$ wget https://github.com/lightningnetwork/lnd/releases/download/v0.4-beta/manifest-v0.4-beta.txt.sig
$ wget https://keybase.io/roasbeef/pgp_keys.asc

$ sha256sum --check manifest-v0.4-beta.txt --ignore-missing
> lnd-linux-arm-v0.4-beta.tar.gz: OK

$ gpg ./pgp_keys.asc
> 65317176B6857F98834EDBE8964EA263DD637C21

$ gpg --import ./pgp_keys.asc
$ gpg --verify manifest-v0.4-beta.txt.sig
> gpg: Good signature from "Olaoluwa Osuntokun <laolu32@gmail.com>" [unknown]
> Primary key fingerprint: 6531 7176 B685 7F98 834E  DBE8 964E A263 DD63 7C21

$ tar -xzf lnd-linux-arm-v0.4-beta.tar.gz
$ ls -la
$ sudo install -m 0755 -o root -g root -t /usr/local/bin lnd-linux-arm-v0.4-beta/*
$ lnd --version
> lnd version 0.4.0-alpha
```

### LND configuration
Now that LND is installed, we need to configure it to work with Bitcoin Core and run automatically on startup.

* Open a "bitcoin" user session  
  `$ sudo su bitcoin` 

* Create the LND working directory and the corresponding symbolic link  
  `$ mkdir /mnt/hdd/lnd`  
  `$ ln -s /mnt/hdd/lnd /home/bitcoin/.lnd`  

* Create the LND configuration file and paste the following content (adjust to your alias). Save and exit.  
  `$ nano /home/bitcoin/.lnd/lnd.conf`

```bash
# RaspiBolt LND Mainnet: lnd configuration
# /home/bitcoin/.lnd/lnd.conf

[Application Options]
debuglevel=debug
debughtlc=true
maxpendingchannels=5
alias=YOUR_NAME [LND]
color=#68F442

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
:point_right: Additional information: [sample-lnd.conf](https://github.com/lightningnetwork/lnd/blob/master/sample-lnd.conf) in the LND project repository

* exit the "bitcoin" user session back to "admin"  
  `$ exit`

* create LND systemd unit and with the following content. Save and exit.  
  `$ sudo nano /etc/systemd/system/lnd.service` 

```bash
# RaspiBolt LND Mainnet: systemd unit for lnd
# /etc/systemd/system/lnd.service

[Unit]
Description=LND Lightning Daemon
Requires=bitcoind.service
After=getpublicip.service
After=bitcoind.service

# for use with sendmail alert
#OnFailure=systemd-sendmail@%n

[Service]
# get var PUBIP from file
EnvironmentFile=/run/publicip

ExecStart=/usr/local/bin/lnd --externalip=${PUBLICIP}
PIDFile=/home/bitcoin/.lnd/lnd.pid
User=bitcoin
Group=bitcoin
Type=simple
KillMode=process
TimeoutSec=180
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```
* enable and start LND  
  `$ sudo systemctl daemon-reload`  
  `$ sudo systemctl enable lnd`  
  `$ sudo systemctl start lnd`  
  `$ systemctl status lnd`  

![Output systemctl status lnd]()ln

* monitor the LND logfile in realtime (exit with `Ctrl-C`)  
  `$ sudo journalctl -f -u lnd`

### LND wallet setup
Once LND is started, the process waits for us to create the integrated Bitcoin wallet (it does not use the bitcoind wallet). 
* start a "bitcoin" user session  
  `$ sudo su bitcoin` 

* Create the LND wallet  
  `$ lncli create` 

* If you want to create a new wallet, enter your `password [C]` as wallet password, select `n` regarding an existing seed and enter the optional `password [D]` as seed passphrase. A new cipher seed consisting of 24 words is created.

![LND new cipher seed]()

These 24 words, combined with your passphrase (optional `password [D]`)  is all that you need to restore your Bitcoin wallet and all Lighting channels. The current state of your channels, however, cannot be recreated from this seed, this requires a continuous backup and is still under development for LND.

:warning: This information must be kept secret at all cost. **Write these 24 words down manually on a piece of paper and store it in a safe place.** This piece of paper is all an attacker needs to completely empty your wallet! Do not store it on a computer. Do not take a picture with your mobile phone. **This information should never be stored anywhere in digital form.**

-----

### Before proceeding to mainnet 
This is the point of no return. Up until now, you can just start over. Experiment with testnet bitcoin. Open and close channels on the testnet. 

Once you switch to mainnet and send real bitcoin to your RaspiBolt, you have "skin in the game". 

* Make sure your RaspiBolt is working as expected.
* Get a little practice with `bitcoin-cli` and its options (see [Bitcoin Core RPC documentation](https://bitcoin-rpc.github.io/))
* Do a dry run with `lncli` and its many options (see [Lightning API reference](http://api.lightning.community/))
* Try a few restarts (`sudo shutdown -r now`), is everything starting fine?

--- 
Next: [Mainnet >>](raspibolt_50_mainnet.md)