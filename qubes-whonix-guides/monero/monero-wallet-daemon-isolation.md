<!---
## Copyright (C) 2018 0xB44EFD8751077F97 <0xB44EFD8751077F97@firemail.cc>
## Email PGP key: 0x4575F28C5441951C8A0056B8CE1A00A773F733E1
## https://github.com/0xB44EFD8751077F97/guides
## See the file COPYING for copying conditions.
-->
# Wallet/Daemon Isolation with Qubes + Whonix
With [Qubes](https://qubes-os.org) + [Whonix](https://whonix.org) you can have a Monero wallet that has no network connection, and runs on a virtually isolated system from the daemon which has all of its traffic forced over [Tor](https://torproject.org).

Qubes gives the flexibility to easily create separate VMs for different purposes. First you will create a Whonix workstation for the wallet with no connection to the network. Next, another Whonix workstation for the daemon which will use a Whonix gateway for networking. For communication between the wallet and daemon you can make use of Qubes [`qrexec`](https://www.qubes-os.org/doc/qrexec3/).

This is safer than other approaches which route the wallet's rpc over a Tor hidden service, or that use physical isolation but still have networking to connect to the daemon. In this way you don't need any network connection on the wallet, you preserve resources of the Tor network, and there is less latency.
## Table of Contents
1. **[Create the TemplateVM and AppVMs](#1-create-templatevm-and-appvms)**
+ 1.1. [Create TemplateVM: `whonix-ws-14-monero`](#11-create-templatevm-whonix-ws-14-monero)
+ 1.2. [Create daemon's AppVM: `monerod-ws`](#12-create-daemons-appvm-monerod-ws)
+ 1.3. [Create wallet's AppVM: `monero-wallet-ws`](#13-create-wallets-appvm-monero-wallet-ws)
+ 1.4. [Create `qrexec` policies](#14-create-qrexec-policies)
2. **[Set Up the TemplateVM](#2-set-up-the-templatevm)**
+ 2.1. [Create system user](#21-create-system-user)
+ 2.2. [Set up `systemd` units](#22-set-up-systemd-units)
  + 2.2.1. [Create mainnet unit](#221-create-mainnet-unit)
  + 2.2.2. [Create stagenet unit](#222-create-stagenet-unit)
  + 2.2.3. [Create testnet unit](#223-create-testnet-unit)
+ 2.3. [Enable the units that should start on boot](#23-enable-the-units-that-should-start-on-boot)
+ 2.4. [Shutdown `whonix-ws-14-monero`](#23-shutdown-whonix-ws-14-monero)
3. **[Set Up the Daemon's AppVM](#3-set-up-the-daemons-appvm)**
+ 3.1. [Get Monero software](#31-get-latest-monero-software)
  + 3.1.1. [Install command-line only tools](#311-install-command-line-only-tools)
  + 3.1.2. [Install GUI tools](#312-install-gui-tools)
+ 3.2. [Create `qrexec` action file](#32-create-qrexec-action-file)
+ 3.3. [Shutdown `monerod-ws`](#33-shutdown-monerod-ws)
4. **[Set Up the Wallet's AppVM](#4-set-up-the-wallets-appvm)**
+ 4.1. [Install wallet binaries](#41-install-wallet-binaries)
+ 4.2. [Create communication channel with daemon on boot](#42-create-communication-channel-with-daemon-on-boot)
+ 4.3. [Shutdown `monero-wallet-ws`](#43-shutdown-monero-wallet-ws)
5. **[Using the Daemon](#5-using-the-daemon)**
+ 5.1. [Control the daemon](#51-control-the-daemon)
+ 5.2. [Monitor the daemon](#52-monitor-the-daemon)
6. **[Using the Command-Line Tools](#6-using-the-command-line-tools)**
+ 6.1. [Getting started](#61-getting-started)
7. **[Using the GUI](#7-using-the-gui)**
+ 7.1. [Launch the GUI](#71-launch-the-gui)
8. **[Advanced Tips](#8-advanced-security-tips)**
+ 8.1. [Enable AppArmor](#81-enable-apparmor)
+ 8.2. [VM hardening](#82-vm-haredening)
+ 8.3. [Qubes `sudo` prompt](#83-dom0-sudo-prompt)

## 1. Create TemplateVM and AppVMs
### **Complete the following commands in a `dom0` terminal.**
### 1.1. Create TemplateVM: `whonix-ws-14-monero`
```
[user@dom0 ~]$ qvm-clone whonix-ws-14 whonix-ws-14-monero
```
### 1.2. Create daemon's AppVM: `monerod-ws`
+ Use a Whonix gateway, typically named `sys-whonix`, for networking.

```
[user@dom0 ~]$ qvm-create --label green --property netvm=sys-whonix --template whonix-ws-14-monero monerod-ws
```
+ Extend the private volume of `monerod-ws` to make space for the blockchain.

```
[user@dom0 ~]$ qvm-volume extend monerod-ws:private 70G
```
+ Enable via `qvm-service`.

```
[user@dom0 ~]$ qvm-service --enable monerod-ws monerod-mainnet
[user@dom0 ~]$ qvm-service --enable monerod-ws monerod-stagenet
[user@dom0 ~]$ qvm-service --enable monerod-ws monerod-testnet
```
### 1.3. Create wallet's AppVM: `monero-wallet-ws`
```
[user@dom0 ~]$ qvm-create --label green --property netvm='' --template whonix-ws-14-monero monero-wallet-ws
```
### 1.4. Create `qrexec` policies
```
[user@dom0 ~]$ echo 'monero-wallet-ws monerod-ws allow' | sudo tee /etc/qubes-rpc/policy/whonix.monerod-{mainnet,stagenet,testnet} >/dev/null
```
## 2. Set Up the TemplateVM
### **Complete the following commands in a `whonix-ws-14-monero` terminal.**
### 2.1. Create system user
```
user@host:~$ sudo useradd --create-home --system --user-group monerod
```
### 2.2. Set up `systemd` units
### 2.2.1. Create mainnet unit
```
user@host:~$ sudo kwrite /lib/systemd/system/monerod-mainnet.service
```
+ Paste the following:

```
[Unit]
Description=Monero Full Node Mainnet
ConditionPathExists=/var/run/qubes-service/monerod-mainnet
After=qubes-sysinit.service

[Service]
User=monerod
Group=monerod
Type=forking
PIDFile=/home/monerod/.bitmonero/monerod.pid
ExecStart=/usr/local/bin/monerod --detach --no-igd --p2p-bind-ip=127.0.0.1 \
    --pidfile=/home/monerod/.bitmonero/monerod.pid
Restart=always

[Install]
WantedBy=multi-user.target
```
### 2.2.2. Create stagenet unit
```
user@host:~$ sudo kwrite /lib/systemd/system/monerod-stagenet.service
```
+ Paste the following:

```
[Unit]
Description=Monero Full Node Stagenet
ConditionPathExists=/var/run/qubes-service/monerod-stagenet
After=qubes-sysinit.service

[Service]
User=monerod
Group=monerod
Type=forking
PIDFile=/home/monerod/.bitmonero/stagenet/monerod.pid
ExecStart=/usr/local/bin/monerod --detach --no-igd --p2p-bind-ip=127.0.0.1 \
    --pidfile=/home/monerod/.bitmonero/stagenet/monerod.pid --stagenet
Restart=always

[Install]
WantedBy=multi-user.target
```
### 2.2.3. Create testnet unit
```
user@host:~$ sudo cp /lib/systemd/system/monerod-stagenet.service /lib/systemd/system/monerod-testnet.service
user@host:~$ sudo sed -i -e 's/stagenet/testnet/g' -e 's/Stagenet/Testnet/g' /lib/systemd/system/monerod-testnet.service
```
+ Fix permissions.

```
user@host:~$ sudo chmod 0644 /lib/systemd/system/monerod-*.service
```
### 2.3. Enable the units that should start on boot
```
user@host:~$ sudo systemctl enable monerod-mainnet.service
user@host:~$ sudo systemctl enable monerod-stagenet.service
user@host:~$ sudo systemctl enable monerod-testnet.service
```
### 2.4. Shutdown `whonix-ws-14-monero`
```
user@host:~$ sudo shutdown now
```
## 3. Set Up the Daemon's AppVM
### **Complete the following commands in a `monerod-ws` terminal.**
### 3.1. Get Monero software
Please note that Monero current version numbers change frequently as the software is updated. In an effort to not give outdated information, this guide will replace the current software current version number with the placeholder `<current version>`.

You should replace `<current version>` with whatever the current version number is. You can get the current version on Monero's [official site](https://getmonero.org/downloads/#linux).
### 3.1.1. Install command-line only tools
+ Use Monero's [guide](https://getmonero.org/resources/user-guides/verification-allos-advanced.html) to download and verify the current software for the command-line only tools.
  + [`Linux, 64-bit Command-Line Tools Only`](https://downloads.getmonero.org/cli/linux64)

+ Extract the software and install the daemon binaries.

```
user@host:~$ tar xf monero-linux-x64-<current version>.tar.bz2 -C ~
user@host:~$ sudo install -g staff -m 0755 -o root ~/monero-<current version>/monero-blockchain-* ~/monero-<current version>/monerod -t /usr/local/bin/
```
+ Copy wallet binaries to their AppVM. Enter `monero-wallet-ws` in the `dom0` prompt.

```
user@host:~$ qvm-copy ~/monero-<current version>/monero-gen-trusted-multisig ~/monero-<current version>/monero-wallet-*
```
### 3.1.2. Install GUI tools
+ Use Monero's [guide](https://getmonero.org/resources/user-guides/verification-allos-advanced.html) to download and verify the current software for the GUI.
  + [`Linux, 64-bit GUI`](https://downloads.getmonero.org/gui/linux64)

+ Extract the software and install the daemon binaries.

```
user@host:~$ tar xf monero-gui-linux-x64-<current version>.tar.bz2 -C ~
user@host:~$ sudo install -g staff -m 0755 -o root ~/monero-gui-<current version>/monero-blockchain-* ~/monero-gui-<current version>/monerod -t /usr/local/bin/
```
+ Copy wallet binaries to their AppVM. Enter `monero-wallet-ws` in the `dom0` prompt.

```
user@host:~$ qvm-copy ~/monero-gui-<current version>/monero-gen-trusted-multisig ~/monero-gui-<current version>/monero-wallet-*
```
### 3.2. Create `qrexec` action files
```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
user@host:~$ echo 'socat STDIO TCP:localhost:18081' | sudo tee /rw/usrlocal/etc/qubes-rpc/whonix.monerod-mainnet >/dev/null
user@host:~$ echo 'socat STDIO TCP:localhost:28081' | sudo tee /rw/usrlocal/etc/qubes-rpc/whonix.monerod-testnet >/dev/null
user@host:~$ echo 'socat STDIO TCP:localhost:38081' | sudo tee /rw/usrlocal/etc/qubes-rpc/whonix.monerod-stagenet >/dev/null
```
+ Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/usrlocal/etc/qubes-rpc/whonix.monerod-*
```
### 3.3. Shutdown `monerod-ws`
```
user@host:~$ sudo shutdown now
```
## 4. Set Up the Wallet's AppVM
### **Complete the following commands in a `monero-wallet-ws` terminal.**
### 4.1. Install wallet binaries
```
user@host:~$ sudo install -g staff -m 0755 -o root ~/QubesIncoming/monerod-ws/monero-gen-trusted-multisig ~/QubesIncoming/monerod-ws/monero-wallet-* -t /usr/local/bin/
```
### 4.2. Create communication channel with daemon on boot
+ Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo kwrite /rw/config/rc.local
```
+ Paste the following on a new line at the bottom of the file:

```
socat TCP-LISTEN:18081,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm monerod-ws whonix.monerod-mainnet" &
socat TCP-LISTEN:28081,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm monerod-ws whonix.monerod-testnet" &
socat TCP-LISTEN:38081,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm monerod-ws whonix.monerod-stagenet" &
```
+ Make the file executable.

```
user@host:~$ sudo chmod +x /rw/config/rc.local
```
### 4.3. Shutdown `monero-wallet-ws`
```
user@host:~$ sudo shutdown now
```
## 5. Using the Daemon
Before using the wallet you must synchronize the blockchain on the AppVM `monerod-ws`. The initial sync can take anywhere from 5 hours to multiple days, depending on your hardware and network.
### **Complete the following commands in a `monerod-ws` terminal.**
### 5.1. Control the daemon
+ Stop or resume blockchain sync at any time by shutting down or starting the AppVM `monerod-ws`, or control the service manually.

```
user@host:~$ sudo systemctl stop monerod-mainnet.service
user@host:~$ sudo systemctl start monerod-mainnet.service
user@host:~$ sudo systemctl status monerod-mainnet.service
```
+ Issue any command to the running daemon as user `monerod`. To see help menu:

```
user@host:~$ sudo -u monerod monerod help
```
### 5.2. Monitor the daemon
+ Check the sync status.

```
user@host:~$ sudo -u monerod monerod status
Height: 1642996/1643496 (99.9%) on mainnet, not mining, net hash 475.35 MH/s, v7, up to date, 5(out)+0(in) connections, uptime 0d 0h 0m 5s
```
+ Watch the debug log.

```
user@host:~$ sudo tail -f /home/monerod/.bitmonero/bitmonero.log
```
+ You may use your wallet when the sync status shows `100.0%`.

```
Height: 1643497/1643497 (100.0%) on mainnet, not mining, net hash 462.02 MH/s, v7, up to date, 8(out)+0(in) connections, uptime 0d 0h 6m 26s
```
+ Or when the debug log shows `SYNCHRONIZED OK`.

```
**********************************************************************
You are now synchronized with the network. You may now start monero-wallet-cli.

Use the "help" command to see the list of available commands.
**********************************************************************
2018-08-21 01:29:15.743 [P2P2]  INFO    global  src/cryptonote_protocol/cryptonote_protocol_handler.inl:1561  SYNCHRONIZED OK
```
## 6. Using the Command-Line Tools
### **Complete the following commands in a `monero-wallet-ws` terminal.**
### 6.1. Getting started
+ To create a new wallet, run `monero-wallet-cli` and follow the instructions.

```
user@host:~$ monero-wallet-cli
```
+ Get help menu from the wallet prompt.

```
[wallet 4xxxxx]: help
```
+ For more examples refer to the [user guide](https://getmonero.org/resources/user-guides/monero-wallet-cli.html).

## 7. Using the GUI
### 7.1. Launch the GUI
```
user@host:~$ monero-wallet-gui
```
## 8. Advanced Tips
### 8.1. Enable AppArmor

+ Use the Whonix wiki for enabling AppArmor: 
  + [https://www.whonix.org/wiki/AppArmor](https://www.whonix.org/wiki/AppArmor)
  + [https://www.whonix.org/wiki/Qubes/AppArmor](https://www.whonix.org/wiki/Qubes/AppArmor)

+ Make use of my experimental Monero profiles:
  + [AppArmor Profiles](https://github.com/0xB44EFD8751077F97/apparmor-profiles/tree/master/apparmor-monero-qubes-whonix)

### 8.2. VM hardening
  + [https://github.com/tasket/Qubes-VM-hardening](https://github.com/tasket/Qubes-VM-hardening)

### 8.3. Qubes `sudo` prompt
  + [https://www.qubes-os.org/doc/vm-sudo/](https://www.qubes-os.org/doc/vm-sudo/)
