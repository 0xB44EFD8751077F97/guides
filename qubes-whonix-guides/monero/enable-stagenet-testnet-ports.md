<!---
## Copyright (C) 2018 0xB44EFD8751077F97 <0xB44EFD8751077F97@firemail.cc>
## Email PGP key: 0x4575F28C5441951C8A0056B8CE1A00A773F733E1
## https://github.com/0xB44EFD8751077F97/
## See the file COPYING for copying conditions.
-->
# Enable Stagenet and Testnet Ports
This is an extension of the Monero user guide "Wallet/Daemon Isolation with Qubes + Whonix". You must complete that guide before this one will work for you.
## Table of Contents
1. **[Set Up `Dom0`](#1-set-up-dom0)**
+ 1.1. [Create `qrexec` policies](#11-create-qrexec-policies)
+ 1.2. [Enable via `qvm-service`](#12-enable-via-qvm-service)
2. **[Set Up the TemplateVM](#2-set-up-the-templatevm)**
+ 2.1. [Create stagenet `systemd` unit](#21-create-stagenet-systemd-unit)
+ 2.2. [Create testnet `systemd` unit](#22-create-testnet-systemd-unit)
+ 2.3. [Enable the units that should start on boot](#23-enable-the-units-that-should-start-on-boot)
3. **[Set Up the Daemon's AppVM](#3-set-up-the-daemons-appvm)**
+ 3.1. [Create `qrexec` action files](#31-create-qrexec-action-files)
+ 3.2. [Shutdown `monerod-ws`](#32-shutdown-monerod-ws)
4. **[Set Up the Wallet's AppVM](#4set-up-the-wallets-appvm)**
+ 4.1. [Create communication channel with daemon on boot](#41-create-communication-channel-with-daemon-on-boot)
+ 4.2. [Shutdown `monero-wallet-ws`](#42-shutdown-monero-wallet-ws)

## 1. Set Up `Dom0`
### 1.1. Create `qrexec` policies
**Complete the following commands in a `dom0` terminal.**
```
[user@dom0 ~]$ sudo tee </etc/qubes-rpc/policy/whonix.monerod-mainnet /etc/qubes-rpc/policy/whonix.monerod-{stagenet,testnet} >/dev/null
```
### 1.2. Enable via `qvm-service`
```
[user@dom0 ~]$ qvm-service --enable monerod-ws monerod-stagenet
[user@dom0 ~]$ qvm-service --enable monerod-ws monerod-testnet
```
## 2. Set Up the TemplateVM
### **Complete the following commands in a `whonix-ws-14-monero` terminal.**
### 2.1. Create stagenet `systemd` unit
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
### 2.2. Create testnet `systemd` unit
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
user@host:~$ sudo systemctl enable monerod-stagenet.service
user@host:~$ sudo systemctl enable monerod-testnet.service
```
## 3. Set Up the Daemon's AppVM
### **Complete the following commands in a `monerod-ws` terminal.**
### 3.1. Create `qrexec` action files
```
user@host:~$ echo 'socat STDIO TCP:localhost:28081' | sudo tee /rw/usrlocal/etc/qubes-rpc/whonix.monerod-testnet >/dev/null
user@host:~$ echo 'socat STDIO TCP:localhost:38081' | sudo tee /rw/usrlocal/etc/qubes-rpc/whonix.monerod-stagenet >/dev/null
```
+ Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/usrlocal/etc/qubes-rpc/whonix.monerod-*
```
### 3.2. Shutdown `monerod-ws`
```
user@host:~$ sudo shutdown now
```
## 4. Set Up the Wallet's AppVM
### **Complete the following commands in a `monero-wallet-ws` terminal.**
### 4.1. Create communication channel with daemon on boot
+ Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo kwrite /rw/config/rc.local
```
+ Paste the following on a new line at the bottom of the file:

```
socat TCP-LISTEN:28081,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm monerod-ws whonix.monerod-testnet" &
socat TCP-LISTEN:38081,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm monerod-ws whonix.monerod-stagenet" &
```
### 4.2. Shutdown `monero-wallet-ws`
```
user@host:~$ sudo shutdown now
```
