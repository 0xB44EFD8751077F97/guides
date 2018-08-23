{% assign version = '2.0.0' | split: '.' %}
{% include disclaimer.html translated="true" version=page.version %}
# Wallet/Daemon Isolation with Qubes + Whonix

With [Qubes](https://qubes-os.org) + [Whonix](https://whonix.org) you can have a Monero wallet that has no network connection, and runs on a virtually isolated system from the daemon which has all of its traffic forced over [Tor](https://torproject.org).

Qubes gives the flexibility to easily create separate VMs for different purposes. First you will create a Whonix workstation for the wallet with no connection to the network. Next, another Whonix workstation for the daemon which will use a Whonix gateway for networking. For communication between the wallet and daemon you can make use of Qubes [qrexec](https://www.qubes-os.org/doc/qrexec3/).

This is safer than other approaches which route the wallet's rpc over a Tor hidden service, or that use physical isolation but still have networking to connect to the daemon. In this way you don't need any network connection on the wallet, you preserve resources of the Tor network, and there is less latency.

Please note that the current version of the Monero software may differ from what these examples show. You should always refer to Monero's [official site](https://getmonero.org/downloads/#linux) for the most recent version.

## Table of contents:

1. **[Create TemplateVM and AppVMs](#1-create-templatevm-and-appvms)**
  + 1.1. [Create TemplateVM](#11-create-templatevm)
  + 1.2. [Create wallet AppVM](#12-create-wallet-appvm)
  + 1.3. [Create daemon AppVM](#13-create-daemon-appvm)
  + 1.4. [Create Qubes qrexec policy](#14-create-qubes-qrexec-policy)
2. **[Set up TemplateVM](#2-set-up-templatevm)**
  + 2.1. [Create system user](#21-create-system-user)
  + 2.2. [Create systemd unit](#22-create-systemd-unit)
  + 2.3. [Shutdown whonix-ws-14-monero](#23-shutdown-whonix-ws-14-monero)
3. **[Set up Daemon AppVM](#3-set-up-daemon-appvm)**
  + 3.1. [Get Monero software](#31-get-latest-monero-software)
  + 3.2. [Create Qubes qrexec action file](#32-create-qubes-qrexec-action-file)
  + 3.3. [Shutdown monerod-ws](#33-shutdown-monerod-ws)
4. **[Set up Wallet AppVM](#4-set-up-wallet-appvm)**
  + 4.1. [Set up wallet binaries](#41-set-up-wallet-binaries)
  + 4.2. [Create communication channel with daemon on boot](#42-create-communication-channel-with-daemon-on-boot)
  + 4.3. [Shutdown monero-wallet-ws](#43-shutdown-monero-wallet-ws)
5. **[Using the Daemon](#5-using-the-daemon)**
  + 5.1. [Control the daemon](#51-control-the-daemon)
  + 5.2. [Monitor the daemon](#52-monitor-the-daemon)
6. **[Using the Command-Line Tools](#6-using-the-command-line-tools)**
  + 6.1. [Getting started](#61-getting-started)
7. **[Using the GUI](#7-using-the-gui)**
  + 7.1. [Getting started](#71-getting-started)
8. **[Advanced Security Tips](#8-advanced-security-tips)**
  + 8.1. [Enable AppArmor](#81-enable-apparmor)
  + 8.2. [VM hardening](#82-vm-haredening)
  + 8.3. [Dom0 sudo prompt](#83-dom0-sudo-prompt)

## 1. Create TemplateVM and AppVMs

**Complete the following commands in a `dom0` terminal.**

### 1.1. Create TemplateVM

+ Clone the TemplateVM `whonix-ws-14`.

```
[user@dom0 ~]$ qvm-clone whonix-ws-14 whonix-ws-14-monero
```

### 1.2. Create wallet AppVM

+ Create an AppVM named `monero-wallet-ws` with no networking.

```
[user@dom0 ~]$ qvm-create --label green --property netvm='' --template whonix-ws-14-monero monero-wallet-ws
```

### 1.3. Create daemon AppVM

+ Create an AppVM named `monerod-ws`. For networking use a Whonix gateway, typically named `sys-whonix`.

```
[user@dom0 ~]$ qvm-create --label green --property netvm=sys-whonix --template whonix-ws-14-monero monerod-ws
```

+ Extend the private volume of the AppVM `monerod-ws` to make space for the blockchain.

```
[user@dom0 ~]$ qvm-volume extend monerod-ws:private 70G
```

+ Enable the `monerod` service in the AppVM `monerod-ws`.

```
[user@dom0 ~]$ qvm-service --enable monerod-ws monerod
```

### 1.4. Create Qubes qrexec policy

+ Create the file `/etc/qubes-rpc/policy/whonix.monerod`.

```
[user@dom0 ~]$ sudo nano /etc/qubes-rpc/policy/whonix.monerod
```

+ Add the following line:

```
monero-wallet-ws monerod-ws allow
```

## 2. Set up TemplateVM

**Complete the following commands in a `whonix-ws-14-monero` terminal.**

### 2.1. Create system user

+ Create a user named `monerod` for running the daemon.

```
user@host:~$ sudo useradd --create-home --system --user-group monerod
```

### 2.2. Create systemd unit

+ Create the file `/lib/systemd/system/monerod.service`.

```
user@host:~$ sudo gedit /lib/systemd/system/monerod.service
```

+ Paste the following:

```
[Unit]
Description=Monero Full Node
ConditionPathExists=/var/run/qubes-service/monerod
After=qubes-sysinit.service

[Service]
User=monerod
Group=monerod
Type=forking
PIDFile=/home/monerod/.bitmonero/monerod.pid
ExecStart=/usr/local/bin/monerod --data-dir=/home/monerod/.bitmonero --detach \
    --log-file=/home/monerod/.bitmonero/bitmonero.log --no-igd \
    --p2p-bind-ip=127.0.0.1 --pidfile=/home/monerod/.bitmonero/monerod.pid
Restart=always

[Install]
WantedBy=multi-user.target
```

+ Enable the unit.

```
user@host:~$ sudo systemctl enable monerod.service
```

### 2.3. Shutdown whonix-ws-14-monero

```
user@host:~$ sudo shutdown now
```

## 3. Set up Daemon AppVM

**Complete the following commands in a `monerod-ws` terminal.**

### 3.1. Get Monero software

+ Use Monero's [guide](https://getmonero.org/resources/user-guides/verification-allos-advanced.html) to download and verify the current software for either the GUI or command-line only tools.
  + [`Linux, 64-bit Command-Line Tools Only`](https://downloads.getmonero.org/cli/linux64)
  + [`Linux, 64-bit GUI`](https://downloads.getmonero.org/gui/linux64)

+ Extract the software, copy the `monerod` and `monero-blockchain-*` binaries to `/usr/local/bin/`, and fix permissions.

```
user@host:~$ tar xf monero-linux-x64-v0.12.3.0.tar.bz2
user@host:~$ sudo cp monero-v0.12.3.0/monerod monero-v0.12.3.0/monero-blockchain-* -t /usr/local/bin/
user@host:~$ sudo chmod 0755 /usr/local/bin/monerod /usr/local/bin/monero-blockchain-*
```

+ Copy `monero-wallet-*` binaries to the AppVM `monero-wallet-ws`.

```
user@host:~$ qvm-copy-to-vm monero-wallet-ws monero-v0.12.3.0/monero-wallet-*
```

### 3.2. Create Qubes qrexec action file

+ Create the file `/rw/usrlocal/etc/qubes-rpc/whonix.monerod`.

```
user@host:~$ sudo mkdir /rw/usrlocal/etc/qubes-rpc
user@host:~$ sudo gedit /rw/usrlocal/etc/qubes-rpc/whonix.monerod
```

+ Add the following line:

```
socat STDIO TCP:localhost:18081
```

### 3.3. Shutdown monerod-ws

```
user@host:~$ sudo shutdown now
```

## 4. Set up Wallet AppVM

**Complete the following commands in a `monero-wallet-ws` terminal.**

### 4.1. Set up wallet binaries

+ Put the binaries in your `$PATH` and fix permissions.

```
user@host:~$ sudo mv ~/QubesIncoming/monerod-ws/monero-wallet-* -t /usr/local/bin/
user@host:~$ sudo chmod 0755 /usr/local/bin/monero-wallet-*
```

### 4.2. Create communication channel with daemon on boot

+ Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo gedit /rw/config/rc.local
```

+ Add the following line to the bottom of the file:

```
socat TCP-LISTEN:18081,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm monerod-ws user.monerod" &
```

+ Make the file executable.

```
user@host:~$ sudo chmod +x /rw/config/rc.local
```

### 4.3. Shutdown monero-wallet-ws

```
user@host:~$ sudo shutdown now
```

## 5. Using the Daemon

Before you can use the wallet you will need to synchronize the Monero blockchain on the AppVM `monerod-ws`. The initial sync can take anywhere from 5 hours to multiple days, depending on your specific hardware and network speeds.

**Complete the following commands in a `monerod-ws` terminal.**

### 5.1. Control the daemon

+ Stop or resume blockchain sync at any time by shutting down or starting the AppVM `monerod-ws`, or control the service manually.

```
user@host:~$ sudo systemctl stop monerod
user@host:~$ sudo systemctl start monerod
user@host:~$ sudo systemctl status monerod
```

+ Issue any command to the running daemon as user `monerod`. To see commands:

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

+ You can begin to use your wallet when the sync status shows:

```
Height: 1643497/1643497 (100.0%) on mainnet, not mining, net hash 462.02 MH/s, v7, up to date, 8(out)+0(in) connections, uptime 0d 0h 6m 26s
```

+ Or when the debug log shows `SYNCHRONIZED OK`:

```
**********************************************************************
You are now synchronized with the network. You may now start monero-wallet-cli.

Use the "help" command to see the list of available commands.
**********************************************************************
2018-08-21 01:29:15.743 [P2P2]  INFO    global  src/cryptonote_protocol/cryptonote_protocol_handler.inl:1561  SYNCHRONIZED OK
```

## 6. Using the Command-Line Tools

Using the monero wallet in the AppVM `monero-wallet-ws` does not require any special commands. For more examples you can refer to Monero's [guide](https://getmonero.org/resources/user-guides/monero-wallet-cli.html) on using the tool `monero-wallet-cli`.

**Complete the following commands in a `monero-wallet-ws` terminal.**

### 6.1. Getting started

+ To create a new wallet, run `monero-wallet-cli` and follow the instructions.

```
user@host:~$ monero-wallet-cli
```

+ Get the help menu from the `monero-wallet-cli` prompt.

```
[wallet 4xxxxx]: help
```

## 7. Using the GUI

### 7.1. Getting started

+ Launch the GUI.

```
user@host:~$ monero-wallet-gui
```

## 8. Advanced Security Tips

### 8.1. Enable AppArmor

+ Use the Whonix wiki for enabling AppArmor: 
  + [https://www.whonix.org/wiki/AppArmor](https://www.whonix.org/wiki/AppArmor)
  + [https://www.whonix.org/wiki/Qubes/AppArmor](https://www.whonix.org/wiki/Qubes/AppArmor)

+ Make use of my Monero profiles:
  + [https://github.com/0xB44EFD8751077F97/apparmor-monero-qubes-whonix](https://github.com/0xB44EFD8751077F97/apparmor-monero-qubes-whonix)

### 8.2. VM hardening

+ Use Tasket's VM hardening tools:
  + [https://github.com/tasket/Qubes-VM-hardening](https://github.com/tasket/Qubes-VM-hardening)

### 8.3. Dom0 sudo prompt

+ Use the Qubes guide for removing passwordless sudo:
  + [https://www.qubes-os.org/doc/vm-sudo/](https://www.qubes-os.org/doc/vm-sudo/)
