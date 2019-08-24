# against-dns-pollution
Defends DNS pollution by the Great Fire Wall using iptables and [unbound](https://nlnetlabs.nl/projects/unbound/about/)

### For queries of A records
Until now, polluted DNS responses have been found to have the following features:
- Will return an A record as long as the question section contains a query for an A record, ignoring other queries.
- Because only one Resource Record (A record) will be returned, the answer section only has a single entry.
- Any garbage enclosed in the additional records will be sent back as-is.
- ipid is always 0.
- "Don't fragment" is always set.

Using the last two features, we can filter out the malicious response that always arrives first with simple iptables rules.
The effect of dropping false-positives is neglectable.

```
sudo iptables -I INPUT -s 8.8.8.8 -p udp --sport 53 -m u32 --u32 "2&0xFFFF=0x0" -j DROP
sudo iptables -I INPUT -s 8.8.8.8 -p udp --sport 53 -m u32 --u32 "4&0xFFFF=0x4000" -j DROP
```

### For other queries
Found by Nick. 
When querying for other records, the GFW's response will add a CNAME record, which pollutes the cache by bringing in a broken A/AAAA record that accompanies the CNAME.

Therefore [unbounded](https://nlnetlabs.nl/projects/unbound/about/) is used here.

> CNAMEs are chased by unbound itself, asking the remote server for every name in the indirection chain, to protect the local cache from illegal indirect referenced items.

```
sudo iptables -I INPUT -s 223.5.5.5 -p udp --sport 53 -m u32 --u32 "32&0xFFFF=0x0001" -j DROP
```

`unbound.conf`:
```
server:
    interface: 127.0.0.1
    interface: ::1
    port: 53
    do-daemonize: no
    prefetch: yes

forward-zone:
    name: "."
    forward-addr: 223.5.5.5
    forward-addr: 8.8.8.8
```

### References
- https://nichi.co/2019/08/12/ip桌子！/
- https://www.usenix.org/system/files/conference/foci14/foci14-anonymous.pdf
