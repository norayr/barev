# Barev File Transfer Protocol Specification v0.1
*Peer-to-Peer File Sharing over Yggdrasil Networks*

---

## Overview

Barev implements file transfers using **XEP-0095 (Stream Initiation)** and **XEP-0096 (SI File Transfer)** protocols adapted for Yggdrasil networks. All transfers are direct peer-to-peer over Yggdrasil IPv6 addresses, with automatic Yggdrasil address selection.

---

## Protocol Flow Summary

```
Sender                              Receiver
  |                                    |
  |-- IQ-set with <si> offer --------->|
  |                                    |
  |<-- IQ-result (accept) -------------|
  |                                    |
  |-- IQ-set with <bytestreams> -------|
  |     (proposed streamhosts)         |
  |                                    |
  |<-- IQ-result (streamhost used) ----|
  |                                    |
  |<-- TCP connection to streamhost ---|
  |     (Receiver connects to Sender)  |
  |                                    |
  |-- File data over socket ---------->|
  |                                    |
  |-- IQ-set (transfer complete) ----->|
  |                                    |
  |<-- IQ-result (ack) ----------------|
```

---

## 1. File Transfer Initiation (XEP-0095)

### Sender Initiates Transfer:
```xml
<iq type="set" id="si_1234567890" to="buddy@buddy_ip" from="you@your_ip">
  <si xmlns="http://jabber.org/protocol/si"
      id="session_abc123"
      mime-type="application/octet-stream"
      profile="http://jabber.org/protocol/si/profile/file-transfer">
    <file xmlns="http://jabber.org/protocol/si/profile/file-transfer"
          name="filename.txt"
          size="1024"/>
    <feature xmlns="http://jabber.org/protocol/feature-neg">
      <x xmlns="jabber:x:data" type="form">
        <field var="stream-method" type="list-single">
          <option><value>http://jabber.org/protocol/bytestreams</value></option>
        </field>
      </x>
    </feature>
  </si>
</iq>
```

### Receiver Accepts Transfer:
```xml
<iq type="result" id="si_1234567890" to="you@your_ip" from="buddy@buddy_ip">
  <si xmlns="http://jabber.org/protocol/si">
    <feature xmlns="http://jabber.org/protocol/feature-neg">
      <x xmlns="jabber:x:data" type="submit">
        <field var="stream-method">
          <value>http://jabber.org/protocol/bytestreams</value>
        </field>
      </x>
    </feature>
  </si>
</iq>
```

### Receiver Rejects Transfer:
```xml
<iq type="error" id="si_1234567890" to="you@your_ip" from="buddy@buddy_ip">
  <error code="403" type="auth">
    <forbidden xmlns="urn:ietf:params:xml:ns:xmpp-stanzas"/>
  </error>
</iq>
```

---

## 2. Bytestreams Negotiation (XEP-0096)

### Sender Proposes Streamhosts:
```xml
<iq type="set" id="bytestream_1234567890" to="buddy@buddy_ip" from="you@your_ip">
  <query xmlns="http://jabber.org/protocol/bytestreams" sid="session_abc123">
    <streamhost jid="you@your_ip" host="your_ygg_ip" port="7890"/>
    <!-- Additional Yggdrasil IPs may be listed -->
    <streamhost jid="you@your_other_ip" host="your_other_ygg_ip" port="7890"/>
  </query>
</iq>
```

**Note**: The `jid` in streamhost MUST match an actual JID known to the receiver (typically the sender's JID).

### Receiver Selects Streamhost:
```xml
<iq type="result" id="bytestream_1234567890" to="you@your_ip" from="buddy@buddy_ip">
  <query xmlns="http://jabber.org/protocol/bytestreams" sid="session_abc123">
    <streamhost-used jid="you@your_ip"/>
  </query>
</iq>
```

---

## 3. Yggdrasil Address Selection

### Algorithm for Choosing Yggdrasil Addresses:

```c
GSList *bonjour_jabber_get_local_ips(int fd) {
    // Returns only Yggdrasil IPv6 addresses (0200::/7 range)
    // Filters out:
    //   - fe80::/10 (link-local)
    //   - fc00::/7 (ULA - fc.. or fd..)
    //   - ::1 (loopback)
    //   - Non-Yggdrasil global IPv6 (2001:... etc.)
}
```

### Yggdrasil Address Recognition:
- First hextet in range `0x0200` to `0x03FF` (hex: `200` to `3ff`)
- Examples: `201:...`, `304:...`, `2ab:...`
- **Not Yggdrasil**: `2001:...` (normal IPv6), `fd00:...` (ULA), `fe80:...` (link-local)

### Address Priority Order:
1. Primary Yggdrasil address (first from `getifaddrs()`)
2. Additional Yggdrasil addresses on other interfaces
3. **Never** includes non-Yggdrasil addresses for file transfer

---

## 4. TCP Stream Setup

### After Streamhost Selection:

1. **Receiver** connects to selected streamhost:
   ```bash
   # Receiver connects to sender's Yggdrasil IP:Port
   nc -6 sender_ygg_ip streamhost_port
   ```

2. **Sender** listens on the advertised port:
   ```c
   // In actual implementation
   int sock = socket(AF_INET6, SOCK_STREAM, 0);
   bind(sock, addr, sizeof(addr));  // addr contains Yggdrasil IP and port
   listen(sock, 1);
   ```

3. **Data Transfer Protocol**:
   - Raw binary data over TCP socket
   - No encapsulation or framing
   - Close socket when transfer complete

---

## 5. Transfer Completion

### Sender Notifies Completion:
```xml
<iq type="set" id="complete_1234567890" to="buddy@buddy_ip" from="you@your_ip">
  <si xmlns="http://jabber.org/protocol/si">
    <feature xmlns="http://jabber.org/protocol/feature-neg">
      <x xmlns="jabber:x:data" type="submit">
        <field var="stream-method">
          <value>http://jabber.org/protocol/bytestreams</value>
        </field>
      </x>
    </feature>
  </si>
</iq>
```

### Receiver Acknowledges:
```xml
<iq type="result" id="complete_1234567890" to="you@your_ip" from="buddy@buddy_ip"/>
```

---

## 6. Error Handling

### Common Error Responses:

#### Invalid Stream Method:
```xml
<iq type="error" id="si_1234567890" to="you@your_ip" from="buddy@buddy_ip">
  <error code="400" type="modify">
    <bad-request xmlns="urn:ietf:params:xml:ns:xmpp-stanzas"/>
    <text xmlns="urn:ietf:params:xml:ns:xmpp-stanzas">
      Unsupported stream method
    </text>
  </error>
</iq>
```

#### No Valid Streamhosts:
```xml
<iq type="error" id="bytestream_1234567890" to="you@your_ip" from="buddy@buddy_ip">
  <error code="404" type="cancel">
    <item-not-found xmlns="urn:ietf:params:xml:ns:xmpp-stanzas"/>
    <text xmlns="urn:ietf:params:xml:ns:xmpp-stanzas">
      No usable streamhosts
    </text>
  </error>
</iq>
```

#### Transfer Cancelled:
```xml
<iq type="error" id="si_1234567890" to="you@your_ip" from="buddy@buddy_ip">
  <error code="403" type="auth">
    <forbidden xmlns="urn:ietf:params:xml:ns:xmpp-stanzas"/>
    <text xmlns="urn:ietf:params:xml:ns:xmpp-stanzas">
      File transfer cancelled by user
    </text>
  </error>
</iq>
```

---

## 7. Implementation Details

### Port Selection:
- Streamhost port is dynamically chosen by sender
- Typically in range 1024-65535
- Must not conflict with listening Jabber port (5299)

### Multiple Yggdrasil Addresses:
If a node has multiple Yggdrasil addresses:
```xml
<streamhost jid="you@primary_ip" host="201:af82:..." port="7890"/>
<streamhost jid="you@secondary_ip" host="201:b570:..." port="7891"/>
```
Receiver tries each until one succeeds.

### Session IDs:
- Must be unique per transfer
- Format: alphanumeric string
- Example: `session_abc123def456`

### Hash Verification (Optional):
```xml
<file name="file.txt" size="1024" hash="md5:abc123..."/>
```
Currently not implemented in Barev but supported by protocol.

---

## 8. Complete Example Session

### Step-by-Step Transfer:

**1. Sender offers file:**
```xml
<iq type="set" id="ft_offer_1" to="buddy@204:7a74:aa1e:101a::a1" from="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7">
  <si xmlns="http://jabber.org/protocol/si"
      id="session_abc123"
      mime-type="image/png"
      profile="http://jabber.org/protocol/si/profile/file-transfer">
    <file xmlns="http://jabber.org/protocol/si/profile/file-transfer"
          name="screenshot.png"
          size="150000"/>
    <feature xmlns="http://jabber.org/protocol/feature-neg">
      <x xmlns="jabber:x:data" type="form">
        <field var="stream-method" type="list-single">
          <option><value>http://jabber.org/protocol/bytestreams</value></option>
        </field>
      </x>
    </feature>
  </si>
</iq>
```

**2. Receiver accepts:**
```xml
<iq type="result" id="ft_offer_1" to="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7" from="buddy@204:7a74:aa1e:101a::a1">
  <si xmlns="http://jabber.org/protocol/si">
    <feature xmlns="http://jabber.org/protocol/feature-neg">
      <x xmlns="jabber:x:data" type="submit">
        <field var="stream-method">
          <value>http://jabber.org/protocol/bytestreams</value>
        </field>
      </x>
    </feature>
  </si>
</iq>
```

**3. Sender proposes streamhosts:**
```xml
<iq type="set" id="streamhost_1" to="buddy@204:7a74:aa1e:101a::a1" from="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7">
  <query xmlns="http://jabber.org/protocol/bytestreams" sid="session_abc123">
    <streamhost jid="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7"
                host="201:af82:9f2f:7809:be0c:360a:1587:6be7"
                port="65432"/>
  </query>
</iq>
```

**4. Receiver selects streamhost:**
```xml
<iq type="result" id="streamhost_1" to="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7" from="buddy@204:7a74:aa1e:101a::a1">
  <query xmlns="http://jabber.org/protocol/bytestreams" sid="session_abc123">
    <streamhost-used jid="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7"/>
  </query>
</iq>
```

**5. Data Transfer:**
```
Receiver connects: nc -6 201:af82:9f2f:7809:be0c:360a:1587:6be7 65432
Sender sends: raw PNG file data (150,000 bytes)
```

**6. Transfer complete:**
```xml
<iq type="set" id="complete_1" to="buddy@204:7a74:aa1e:101a::a1" from="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7">
  <si xmlns="http://jabber.org/protocol/si">
    <feature xmlns="http://jabber.org/protocol/feature-neg">
      <x xmlns="jabber:x:data" type="submit">
        <field var="stream-method">
          <value>http://jabber.org/protocol/bytestreams</value>
        </field>
      </x>
    </feature>
  </si>
</iq>

<iq type="result" id="complete_1" to="inky@201:af82:9f2f:7809:be0c:360a:1587:6be7" from="buddy@204:7a74:aa1e:301a::a1"/>
```

---

## 9. Security Considerations

### Trust Model:
- Direct peer-to-peer only (no proxies)
- Both ends must have valid Yggdrasil connectivity
- No encryption of file data (plain TCP)
- JID must match actual connection IP

### Validation:
1. Streamhost `jid` must match sender's JID
2. Streamhost `host` must be a valid Yggdrasil address
3. Receiver validates address is in Yggdrasil range (0200::/7)

### Limitations:
- No resume capability
- No parallel transfers (one at a time per buddy)
- Maximum file size limited by available memory
- No progress reporting during transfer

---

## 10. Implementation Notes for Client Developers

### Required Functions:

#### 1. Yggdrasil Address Detection:
```python
def is_yggdrasil_addr(ip):
    """Check if IPv6 address is in Yggdrasil range (0200::/7)"""
    if ':' not in ip:
        return False
    first_hextet = ip.split(':')[0]
    try:
        val = int(first_hextet, 16)
        return 0x200 <= val <= 0x3ff
    except:
        return False
```

#### 2. Streamhost Selection:
```python
def select_streamhost(streamhosts):
    """Choose the best streamhost from list"""
    for host in streamhosts:
        if is_yggdrasil_addr(host['host']):
            return host
    return None  # No valid Yggdrasil streamhost
```

#### 3. File Transfer State Machine:
```python
states = {
    'OFFER_SENT': 1,
    'ACCEPTED': 2,
    'STREAMHOST_PROPOSED': 3,
    'STREAMHOST_SELECTED': 4,
    'TRANSFERRING': 5,
    'COMPLETE': 6,
    'ERROR': 7
}
```

### Testing File Transfers:

#### Manual Test with Netcat:
```bash
# Sender (listen)
nc -6 -l -p 65432 > received_file

# Receiver (connect and send)
cat file_to_send | nc -6 sender_ygg_ip 65432
```

---

## 11. Troubleshooting

### Common Issues:

| Problem | Solution |
|---------|----------|
| "No usable streamhosts" | Ensure sender has Yggdrasil IPs in `bonjour_jabber_get_local_ips()` |
| Connection refused | Check firewall allows inbound connections on streamhost port |
| Transfer hangs | Verify both ends have Yggdrasil connectivity (`yggdrasilctl getPeers`) |
| Wrong IP in streamhost | JID IP must match actual connection IP |
| Transfer too slow | Check Yggdrasil routing, use `yggdrasilctl getSwitchPeers` |

### Debug Commands:
```bash
# Check Yggdrasil status
yggdrasilctl status
yggdrasilctl getPeers

# Monitor file transfer ports
sudo netstat -tlnp | grep 5299
sudo netstat -tlnp | grep 65432  # or whatever streamhost port

# Test connectivity
ping6 buddy_ygg_ip
telnet buddy_ygg_ip 5299
```

---

## 12. Reference Implementation

See Barev source code:
- `bonjour_ft.c` - Main file transfer implementation
- `jabber.c` - `bonjour_jabber_get_local_ips()` function
- `xep_bytestreams_parse()` - Streamhost negotiation
- `xep_si_parse()` - Stream initiation handling

---

*Last Updated: 2025-12-11*
*Protocol Version: 0.1*
*Based on XEP-0095 and XEP-0096 with Yggdrasil adaptations*
