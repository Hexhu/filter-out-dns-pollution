# anti-dns-pollution
Defends DNS pollution by the Great Fire Wall using [iptables](https://wiki.archlinux.org/index.php/Iptables) and [unbound](https://nlnetlabs.nl/projects/unbound/about/)

### Protect queries for A records
Until now, polluted DNS responses have been found to have the following characteristics:
- Will return just an A record as long as the question section contains an entry for A record. All other queries are ignored.
- Because only one Resource Record (A record) will be returned, the answer section only has that single, polluted entry.
- Any garbage enclosed in the additional records will be sent back as-is.
- ipid is always 0.
- "Don't fragment" is always set.

Using the last two features, we can distinguish [polluted (hijacked)](https://en.wikipedia.org/wiki/DNS_hijacking) responses from legitimate ones, which will eventually arrive, because 
We can filter out the malicious responses with simple iptables rules.

```
iptables -I INPUT -s <DNS_server_name> -p udp --sport 53 -m u32 --u32 "2&0xFFFF=0x0" -j DROP
iptables -I INPUT -s <DNS_server_name> -p udp --sport 53 -m u32 --u32 "4&0xFFFF=0x4000" -j DROP
iptables -I INPUT -s <DNS_server_name> -p udp --sport 53 -m u32 --u32 "32&0xFFFF=0x0001" -j DROP
```

Replace <DNS_server_name> with your favorite caching DNS server (1.1.1.1, 8.8.8.8, 1.1, etc.).

Enough to handle most cases.

The effect of dropping false-positives is neglectable.

### Protect other queries
Credit to [Nick](https://gitlab.com/NickCao). 

When querying for other types of records, GFW's responses will contain a CNAME record, which pollutes local cache (the one used by the DNS server on your home router, for example) by attaching in a broken A/AAAA record with the CNAME.

Therefore, we employ [unbound](https://nlnetlabs.nl/projects/unbound/about/) to properly handle CNAME record lookup.

> CNAMEs are traced by unbound itself, asking the remote server for every name in the indirection chain, to protect the local cache from illegal indirect referenced items.

```
sudo iptables -I INPUT -s <DNS_server_name> -p udp --sport 53 -m u32 --u32 "32&0xFFFF=0x0001" -j DROP
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
    forward-addr: <DNS_server_name#1>
    forward-addr: <DNS_server_name#2>
```

Again, replace <DNS_server_name> with your favorite caching DNS server.

### TODO

- [ ] Monitor IPv6 DNS traffic to see if the above still applies.

### References
- https://nichi.co/posts/2019-08-12-ip%E6%A1%8C%E5%AD%90/
- https://www.usenix.org/system/files/conference/foci14/foci14-anonymous.pdf
- https://netfilter.org/projects/iptables/
- https://nlnetlabs.nl/projects/unbound/about/
