IP Spoofing Attack & Prevention Lab (Portfolio Write‑Up)
Demonstrating source IP spoofing with iptables SNAT and mitigation with Cisco Extended ACLs

Quick Facts
•	Environment: Isolated lab network (no production systems).
•	Attacker host: AlpineLinux‑1 on Network A (192.168.2.0/24).
•	Victim host: AlpineLinux‑2 on Network B (200.41.3.0/24).
•	Router: Cisco R1 with FastEthernet0/0 (Network A) and FastEthernet0/1 (Network B).
•	IDS view: Wireshark capture to validate packet headers.
•	IPS control: Cisco Extended ACL applied on the router to block spoofed sources.

1) Overview
This lab demonstrates a classic network-layer issue: an IP packet’s source address can be forged if the network does not enforce validation. In the first half, the attacker host intentionally changes the source IP address of outgoing ICMP (ping) traffic using iptables SNAT in the NAT table. In the second half, the router is configured with an Extended ACL that only permits traffic from the legitimate attacker subnet and drops everything else, blocking the spoofed packets. The goal is to clearly show the “attack → observe → mitigate → re-test” lifecycle.

2) What IP Spoofing Means (in practical terms)
IP spoofing is when a sender forges the Source IP field in an IP header so the packet appears to come from a different machine. Routers forward packets based on destination routing, not on whether the source address is truthful. If the network lacks controls like anti-spoofing ACLs (or uRPF), spoofed packets can travel across segments and reach the target.
Common reasons attackers use spoofing:
•	To hide the real origin of traffic (basic anonymity at the IP layer).
•	To bypass weak IP-based trust rules (e.g., “allow from internal range”).
•	To support reflection/amplification attacks (spoofing the victim’s IP).

3) Lab Topology and Addressing
The lab uses two LANs connected by a single Cisco router:
AlpineLinux‑1 (Attacker)  →  Switch1  →  R1 (Router)  →  Switch2  →  AlpineLinux‑2 (Victim)

Network segments:
Network	Purpose	Subnet
Network A	Attacker side LAN	192.168.2.0/24
Network B	Victim side LAN	200.41.3.0/24
Router interface IPs (R1):
•	FastEthernet0/0 (toward Network A): 192.168.2.1/24
•	FastEthernet0/1 (toward Network B): 200.41.3.1/24

4) Initial System Setup (Alpine Linux)
Alpine Linux is minimal and does not ship with iptables by default, so iptables was installed on both attacker and victim hosts to enable packet filtering/NAT features.
Commands used (run as root):
apk add iptables
iptables -L

5) IP Address Configuration
Static IPs were assigned so routing and ACL behavior would be predictable. On Alpine, this was done by editing /etc/network/interfaces.
Attacker (AlpineLinux‑1) – /etc/network/interfaces:
address 192.168.2.2
netmask 255.255.255.0
gateway 192.168.2.1
Victim (AlpineLinux‑2) – /etc/network/interfaces:
address 200.41.3.2
netmask 255.255.255.0
gateway 200.41.3.1

Router R1 interface configuration (Cisco IOS):
interface f0/0
 ip address 192.168.2.1 255.255.255.0
!
interface f0/1
 ip address 200.41.3.1 255.255.255.0

6) Phase 1 — Baseline Connectivity (before spoofing)
Before any manipulation, normal connectivity was verified end‑to‑end. From AlpineLinux‑1, a standard ping to the victim confirmed that routing was working and that the router was forwarding ICMP normally.
From attacker:
ping 200.41.3.2
Expected result: ICMP echo replies are received. In this baseline state, the packet header shows Source = 192.168.2.2 and Destination = 200.41.3.2.
📸 Screenshot placement suggestion: Baseline ping success and/or Wireshark capture showing Source 192.168.2.2 → Dest 200.41.3.2.

7) Phase 2 — IP Spoofing with iptables SNAT (Attack)
The key technique used here is Source NAT (SNAT) in the POSTROUTING chain. POSTROUTING happens right before packets leave the host, which makes it an ideal place to rewrite the source address. Instead of crafting packets manually, SNAT lets us demonstrate spoofing using a normal ping command while iptables transparently modifies the outgoing ICMP packets.
Spoof rule applied on AlpineLinux‑1 (attacker):
iptables -t nat -A POSTROUTING -p icmp -j SNAT --to-source 246.79.20
What each part does:
•	-t nat: use the NAT table (address translation).
•	POSTROUTING: modify packets just before they leave the host.
•	-p icmp: apply this rule only to ICMP traffic (ping).
•	-j SNAT: perform Source NAT (rewrite the source IP).
•	--to-source 246.79.20: set the new (forged) source IP address.
Then the attacker runs a normal ping again:
ping 200.41.3.2
Observation: On the wire (as seen by Wireshark on the router or victim side), the ICMP request now shows Source IP = 246.79.20 and Destination = 200.41.3.2. That proves the spoofing worked: the victim and router receive a packet that claims it came from an unrelated address.
📸 Screenshot placement suggestion: Wireshark capture showing ICMP with Source IP = 246.79.20 → Dest 200.41.3.2.

8) Phase 3 — Detection (IDS view with Wireshark)
Wireshark acts as the IDS perspective in this lab. The main detection idea is simple: compare what the network expects (traffic from 192.168.2.0/24 arriving from the attacker-side) versus what is actually observed (a packet arriving from the attacker side that claims a completely different source IP).
Useful Wireshark filter:
icmp
Key indicator: the ICMP packets are physically coming from the attacker segment path, yet the Source IP in the IP header is not from the attacker subnet. That mismatch is the practical signature of spoofing in this setup.

9) Phase 4 — Prevention (IPS control with Cisco Extended ACL)
To prevent spoofing, the router enforces a rule: traffic that originated from the attacker LAN must have a source address in 192.168.2.0/24. Any packet with a different source address is treated as spoofed and dropped. This is a classic anti‑spoofing control at the network boundary.
ACL used (Extended ACL 110):
access-list 110 permit ip 192.168.2.0 0.0.0.255 any
access-list 110 deny ip any any
Explanation: The first line permits only traffic whose source is in 192.168.2.0/24. The second line denies everything else, which includes the spoofed packets claiming to originate from 246.79.20.
Applying the ACL to the router interface (as done in the report):
interface f0/1
 ip access-group 110 out
📸 Screenshot placement suggestion: Router configuration showing ACL 110 applied to f0/1 (outbound).

10) Phase 5 — Re-test and Proof of Blocking
With the ACL in place, the spoofed ping is attempted again. The router drops the spoofed traffic, and Wireshark should show no spoofed ICMP requests reaching the victim.
Verification command on the router:
show access-lists 110
In the report, the deny hit counter increases (example: “deny 2830 matches”), which is strong evidence that the ACL is actively blocking packets that do not match the permitted source range.
📸 Screenshot placement suggestion: Output of “show access-lists 110” showing deny match count increasing (report image on page 5).

11) Cleanup — Removing the Spoofing Rule (return to normal traffic)
Finally, the iptables SNAT rule is removed from the attacker to restore normal source addressing. Once removed, ping traffic originates from the legitimate 192.168.2.2 again.
Remove the SNAT rule on the attacker:
iptables -t nat -D POSTROUTING -p icmp -j SNAT --to-source 246.79.20

12) Key Takeaways (Network Infrastructure Perspective)
•	IP headers can be forged; the network will still route packets unless anti‑spoofing controls exist.
•	SNAT in POSTROUTING is an easy way to demonstrate spoofing without custom packet crafting.
•	Wireshark helps validate what is actually on the wire (truth) vs. what hosts assume (trust).
•	Extended ACLs at the router boundary are an effective first-line defense to enforce “source must match ingress network.”

Disclaimer
This lab was performed in an isolated environment for educational and defensive security learning only. Do not attempt spoofing techniques on networks you do not own or explicitly have permission to test.
