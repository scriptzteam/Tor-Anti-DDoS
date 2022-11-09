# Tor-Anti-DDoS
Attempt against the ongoing Tor DDoS attacks - to block unwanted traffic to relays and (hopefully) help against the ongoing Tor DDoS attacks.

## How does it work?
The rules shown here make use of a mix of the `recent` and `hashlimit` iptables modules. Should an attacker hit 5 SYNs in one second on the ORPort the IP is blocked for 60 seconds. Should another SYN attempt arrive in that timeframe the timer is reset and the IP stays blocked for another 60 seconds.

In addition to that, there are no more than 5 connections allowed per source ip to the Tor port.

## How well does it work?
Very well in my observations. Before the rules were in place I had many of the infamous "Your computer is too slow to handle this many circuit creation requests" in my log. After both my relays lost their `Stable`, `Guard` and `HSDir` flags I finally decided to do something against it (and you should too if you are a relay operator).

Since the rules are active, directory authorities are happy again and my relays have their flags back. The infamous log message is gone. Additionally the behaviour of the tor processes are back to pre-DDoS times, both in terms of traffic and on strain on CPU and memory.

## Whitelisting the directory authorities and snowflakes.
Since we always want to allow directory authorities and snowflakes to be able to talk to our relay we always `ACCEPT` them before attempting to ratelimit. To get the addresses of these you can use the following commands. The addresses should very rarely change, if ever. You see these addresses used beneath in the actual ruleset.

for v4:
```
$ getent ahostsv4 snowflake-01.torproject.net. | awk '{ print $1 }' | sort -u | xargs
$ curl -s 'https://onionoo.torproject.org/summary?search=flag:authority' -o - | jq -cr '.relays[].a[0]' | sort | xargs
```

for v6:
```
$ getent ahostsv6 snowflake-01.torproject.net. | awk '{ print $1 }' | sort -u | xargs
$ curl -s 'https://onionoo.torproject.org/summary?search=flag:authority' -o - | jq -cr '.relays[].a | select(length > 1) | .[1]' | tr -d '][' | sort | xargs
```

## The actual rules
This are the actual iptables rules, trimmed down to only the most relevant parts. I left out ip6tables for an exercise to the reader since it's basically the same.

Of course you must change `$DSTIP` and `$DSTPORT` for your environment.

```
iptables -I INPUT -m state --state INVALID -j DROP
iptables -I INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -I INPUT -i lo -j ACCEPT
iptables -I INPUT -p icmp -j ACCEPT
iptables -I INPUT -s 128.31.0.34/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 131.188.40.189/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 154.35.175.225/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 171.25.193.9/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 193.23.244.244/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 199.58.81.140/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 204.13.164.118/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 45.66.33.45/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 66.111.2.131/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 86.59.21.38/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I INPUT -s 193.187.88.42/32 -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -N TOR_RATELIMIT
iptables -I INPUT -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -m state --state NEW -j TOR_RATELIMIT
iptables -I INPUT -d $DSTIP/32 -p tcp -m tcp --dport $DSTPORT -j ACCEPT
iptables -I OUTPUT -j ACCEPT
iptables -I TOR_RATELIMIT -m connlimit --connlimit-above 5 --connlimit-mask 32 --connlimit-saddr -j DROP
iptables -I TOR_RATELIMIT -m recent --update --seconds 60 --name tor-recent --mask 255.255.255.255 --rsource -j DROP
iptables -I TOR_RATELIMIT -m hashlimit --hashlimit-upto 5/sec --hashlimit-burst 5 --hashlimit-mode srcip --hashlimit-name tor-hashlimit -j RETURN
iptables -I TOR_RATELIMIT -m recent --set --name tor-recent --mask 255.255.255.255 --rsource
iptables -I TOR_RATELIMIT -j DROP
```
