# 🛡️ IP Spoofing Attack & Prevention Lab

**Demonstrating source IP spoofing using iptables SNAT and mitigation with Cisco Extended ACLs**

[![Lab Type](https://img.shields.io/badge/Lab-Network%20Security-blue)]()
[![Platform](https://img.shields.io/badge/Platform-Cisco%20%7C%20Alpine%20Linux-orange)]()
[![Status](https://img.shields.io/badge/Status-Complete-success)]()

---

## 🔎 Overview

This lab demonstrates a classic **network-layer security weakness**: IP source addresses can be forged if the network does not enforce validation. The project follows a complete **attack → observe → mitigate → verify** lifecycle to showcase both offensive and defensive security techniques.

### What You'll Learn

- How IP spoofing works at the network layer
- Using iptables SNAT for traffic manipulation
- Packet analysis with Wireshark
- Implementing Cisco Extended ACLs for anti-spoofing
- Network boundary security controls

### Lab Environment

- **Attacker Host:** AlpineLinux-1 (Network A – `192.168.2.0/24`)
- **Victim Host:** AlpineLinux-2 (Network B – `200.41.3.0/24`)
- **Router:** Cisco R1
- **Detection:** Wireshark packet capture
- **Prevention:** Cisco Extended ACL

---

## 🌐 Lab Topology

```
AlpineLinux-1 (Attacker)          AlpineLinux-2 (Victim)
192.168.2.2                        200.41.3.2
      |                                  |
  Switch-1                           Switch-2
      |                                  |
      └────────── Router R1 ─────────────┘
              (Fa0/0)    (Fa0/1)
           192.168.2.1  200.41.3.1
```

### Network Segments

| Network    | Purpose     | Subnet            | Gateway       |
|------------|-------------|-------------------|---------------|
| Network A  | Attacker LAN | `192.168.2.0/24` | `192.168.2.1` |
| Network B  | Victim LAN   | `200.41.3.0/24`  | `200.41.3.1`  |

---

## 🛠️ Technologies Used

- **Alpine Linux** - Lightweight Linux distribution for hosts
- **iptables** - Linux packet filtering and NAT
- **Cisco IOS** - Router operating system
- **Wireshark** - Network protocol analyzer
- **ICMP** - Test protocol for demonstration

---

## 📖 Lab Phases

### Phase 1: Environment Setup

Install required packages on Alpine Linux:

```bash
apk add iptables
iptables -L
```

Configure IP addressing:

**Attacker (AlpineLinux-1):**
```plaintext
IP Address : 192.168.2.2
Netmask    : 255.255.255.0
Gateway    : 192.168.2.1
```

**Victim (AlpineLinux-2):**
```plaintext
IP Address : 200.41.3.2
Netmask    : 255.255.255.0
Gateway    : 200.41.3.1
```

**Router R1 (Cisco IOS):**
```cisco
interface FastEthernet0/0
 ip address 192.168.2.1 255.255.255.0
 
interface FastEthernet0/1
 ip address 200.41.3.1 255.255.255.0
```

### Phase 2: Baseline Connectivity Test

Verify normal connectivity before attack:

```bash
ping 200.41.3.2
```

**Expected Result:** ICMP echo replies received with legitimate source IP (`192.168.2.2`)

---

### Phase 3: IP Spoofing Attack (SNAT)

Apply Source NAT rule to forge packets:

```bash
iptables -t nat -A POSTROUTING -p icmp -j SNAT --to-source 246.79.20
```

**Rule Breakdown:**
- `-t nat` → NAT table
- `POSTROUTING` → Modify packets before egress
- `-p icmp` → Apply to ICMP traffic only
- `SNAT --to-source 246.79.20` → Rewrite source IP to spoofed address

Execute the attack:

```bash
ping 200.41.3.2
```

**Result:** Victim receives packets appearing to originate from `246.79.20` instead of the real source.

---

### Phase 4: Detection (IDS View)

Capture and analyze traffic using Wireshark:

**Display Filter:**
```plaintext
icmp
```

**Detection Indicators:**
- Packet arrives from attacker-side interface
- Source IP (`246.79.20`) does not match attacker subnet (`192.168.2.0/24`)
- Logical impossibility based on network topology

---

### Phase 5: Prevention (IPS Implementation)

Deploy Cisco Extended ACL to enforce source validation:

```cisco
access-list 110 permit ip 192.168.2.0 0.0.0.255 any
access-list 110 deny ip any any

interface FastEthernet0/0
 ip access-group 110 in
```

**ACL Logic:**
- Only permit traffic from Network A with valid source IPs (`192.168.2.0/24`)
- Deny all other traffic (including spoofed packets)

---

### Phase 6: Verification & Testing

Re-execute the spoofed ping:

```bash
ping 200.41.3.2
```

**Expected Result:** No replies received (spoofed traffic blocked at router)

Verify ACL effectiveness:

```cisco
show access-lists 110
```

**Expected Output:** Deny counter increments, proving mitigation is active.

---

### Phase 7: Cleanup

Remove spoofing rule to restore normal traffic:

```bash
iptables -t nat -D POSTROUTING -p icmp -j SNAT --to-source 246.79.20
```

Verify legitimate connectivity restored:

```bash
ping 200.41.3.2
```

---

## 🎓 Key Takeaways

### Technical Insights

1. **IP addresses are not authenticated by default** - Routers forward based on destination, not source validity
2. **SNAT enables realistic attack simulation** - Demonstrates how easily source IPs can be forged
3. **Wireshark reveals the truth** - Packet captures show what's actually on the wire
4. **Network boundary controls are critical** - ACLs provide effective anti-spoofing defense
5. **Defense-in-depth matters** - Multiple layers of validation strengthen security posture

### Security Principles Demonstrated

- **Attack Surface Understanding** - Knowing how attacks work enables better defense
- **Detection vs Prevention** - IDS (Wireshark) observes; IPS (ACL) blocks
- **Ingress Filtering** - Validating source addresses at network entry points
- **Security by Design** - Proactive controls prevent exploitation

---

## 📚 Related Concepts

This lab connects to several important security topics:

- **BCP 38 / RFC 2827** - Network Ingress Filtering best practices
- **Unicast Reverse Path Forwarding (uRPF)** - Advanced anti-spoofing technique
- **DDoS Reflection Attacks** - How spoofing enables amplification
- **TCP Sequence Prediction** - Historical blind spoofing attacks

---

## 🔧 Potential Extensions

Want to expand this lab? Consider:

- [ ] Implement Unicast RPF as alternative to ACLs
- [ ] Test TCP spoofing (more complex due to state)
- [ ] Create automated detection scripts
- [ ] Simulate DDoS reflection scenario
- [ ] Add logging and alerting mechanisms

---

## ⚠️ Disclaimer

**This lab was conducted in an isolated environment for educational and defensive security learning only.**

**DO NOT** attempt IP spoofing on networks you do not own or have explicit written permission to test. Unauthorized network manipulation may violate:

- Computer Fraud and Abuse Act (CFAA)
- Network usage policies
- Service provider terms of service
- Local and federal laws

This project is documented as a **technical portfolio demonstration**, not for malicious purposes.

---

## 📝 License

This project is available under the MIT License for educational purposes.

---

## 🤝 Connect

Questions or suggestions? Feel free to open an issue or reach out!

**Author:** [Vibhav Chennamadhava]  
**LinkedIn:** [https://www.linkedin.com/in/vibhav-chennamadhava-61184b190/]  
**Portfolio:** [https://vibhavchennamadhava.github.io/]

---

*Built with 🔒 for learning network security fundamentals*

