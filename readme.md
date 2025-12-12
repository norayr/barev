# Barev — Bonjour like Pidgin/libpurple plugin & protocol for Yggdrasil network

**Barev** was started initially as a fork of Pidgin’s Bonjour protocol plugin, adapted for **manual peer addressing** (no mDNS / no Avahi required).
It’s meant for direct peer-to-peer chats (and related features) over IPv6—commonly used with **Yggdrasil**.

Before making **Barev**, the [attempt](https://github.com/norayr/ybb) of using **Bonjour** over Yggdrasil was made. It works, but is limited. Thus this project.

# Compile

## Requirements

You need Pidgin (libpurple) development headers and common build tools.

## Dependencies

On Gentoo: net-im/pidgin, dev-libs/libxml2

On Debian: libpurple-dev, libxml2-dev

## Actual build

```
git clone https://github.com/norayr/barev
cd barev
make
```

then
```
sudo make install
```
or just manually copy libbarev.so to ~/.purple/plugins
```
cp libbarev.so ~/.purple/plugins
```

restart pidgin.

# Usage
Because Barev does not do discovery, you must exchange contact details out-of-band.

Each side provides the other party their
* nick
* yggdrasil ip
* port (on which barev is listening)

Add your buddy (Buddies -> Add Buddy).

Enter the peer's nick@ipv6

The contact will be added to ` ~/.purple/barev-contacts-<nick>.txt`

Currently, if you want to change your contact's port, you need to

* Stop pidgin
* Edit that file, change port
* Start pidgin

# What works

* p2p messages
* file transfers
* statuses: available, away, do not disturb, status lines.
* avatars (XEP-0153)

# What does not work

but nice to have

avatars, audio/video.
no mucs.

# Dreams

XMPP publishing.
