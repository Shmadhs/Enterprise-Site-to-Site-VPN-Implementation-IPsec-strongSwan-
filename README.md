# Enterprise Site-to-Site VPN Implementation (IPsec & strongSwan)

## Project Overview
This project demonstrates the design and deployment of a secure, encrypted Site-to-Site Virtual Private Network (VPN) using **IPsec** and **strongSwan**. The architecture connects two isolated local area networks (LANs) across an untrusted network, enforcing strong mutual authentication via a custom Public Key Infrastructure (PKI) and encapsulating all transit data using the Encapsulating Security Payload (ESP) protocol.

**Technologies Used:** Linux, IPsec, strongSwan (`swanctl`), OpenSSL (PKI/RSA 4096-bit), TCP/IP Routing, Wireshark (Traffic Analysis).

---

## Network Topology & Routing Architecture
The environment consists of four Virtual Machines: two acting as VPN Gateways and two acting as internal corporate hosts.

* **Gateway 1:** External IP: `10.0.3.3` | Internal IP: `192.168.1.1/24`
* **Gateway 2:** External IP: `10.0.3.4` | Internal IP: `192.168.2.1/24`
* **Internal Host 1:** IP: `192.168.1.10` (Default GW: `192.168.1.1`)
* **Internal Host 2:** IP: `192.168.2.10` (Default GW: `192.168.2.1`)

### Layer 3 Routing Setup
To establish base connectivity before tunnel negotiation, IP forwarding was enabled on both gateways, and static routes were configured to point to the opposing gateway's external interface.

```bash
# Enable IP Forwarding on Gateways
sudo sysctl -w net.ipv4.ip_forward=1

# Gateway 1 Route to Gateway 2's subnet
sudo ip route add 192.168.2.0/24 via 10.0.3.4 dev eth0

# Gateway 2 Route to Gateway 1's subnet
sudo ip route add 192.168.1.0/24 via 10.0.3.3 dev eth0
```
Phase 1: Public Key Infrastructure (PKI) & Authentication
To maximize security against brute-force attacks and avoid the vulnerabilities of Pre-Shared Keys (PSKs), mutual authentication was enforced using digital certificates. An internal Certificate Authority (CA) was established to act as the "Trust Anchor."
```bash
# Generate CA Root Key and Self-Signed Certificate
openssl genrsa -out caKey.pem 4096 
openssl req -x509 -new -nodes -key caKey.pem -days 3650 -out caCert.pem -subj "/C=GR/O=Lab/CN=VPN CA"

# Generate 4096-bit RSA Key and CSR for Gateway 1
openssl genrsa -out g1Key.pem 4096
openssl req -new -key g1Key.pem -out gateway1.csr -subj "/C=GR/O=Lab/CN=gateway1"

# CA signs Gateway 1's Certificate
openssl x509 -req -in gateway1.csr -CA caCert.pem -CAkey caKey.pem -CAcreateserial -out g1Cert.pem -days 730
```
(This process was mirrored for Gateway 2. The resulting keys and certificates were securely placed in the respective /etc/swanctl/ directories with strict chmod 600 permissions).

Phase 2: IPsec & strongSwan Configuration (swanctl.conf)
The swanctl.conf file was configured to establish the Security Associations (SAs).

IKEv2 Authentication: Enforced via auth = pubkey, matching the exact Distinguished Name (DN) issued by the OpenSSL CA.

On-Demand Negotiation: The start_action = trap policy was utilized. The VPN does not stay permanently active; instead, the kernel monitors the traffic and automatically dynamically raises the IPsec tunnel only when traffic matching the defined selectors (192.168.1.0/24 <-> 192.168.2.0/24) is detected.

Gateway 1 Configuration Extract:
```bash
connections { 
   gw1-to-gw2 { 
      remote_addrs = 10.0.3.4 
      local { 
         auth = pubkey 
         certs = g1Cert.pem 
      } 
      remote { 
         auth = pubkey 
         id = "C=GR, O=Lab, CN=gateway2" 
      } 
      children { 
         net { 
            local_ts = 192.168.1.0/24 
            remote_ts = 192.168.2.0/24 
            start_action = trap 
         } 
      } 
   } 
}
```
Phase 3: Tunnel Verification & Traffic Analysis
The configuration was loaded into the strongSwan daemon seamlessly. To trigger the trap policy, an ICMP ping was initiated from Host 1 to Host 2.
```bash
# Load certificates and configuration
sudo swanctl --load-all  

# Trigger the IPsec tunnel creation
ping -c 3 192.168.2.1

# Verify Security Associations
sudo swanctl --list-sas
```
### Wireshark ESP & SPI Verification
To mathematically prove the security of the tunnel, network traffic was captured via Wireshark on the Gateway's external interface during the Ping test.

Findings:

No Cleartext Data: Standard ICMP packets were completely absent from the external network segment.

ESP Encapsulation: All captured traffic was strictly Encapsulating Security Payload (ESP). The payload consisted of randomized, unreadable bytes, verifying active AES encryption.

SPI Matching: The Security Parameter Index (SPI) values captured in Wireshark perfectly matched the SPI values reported by swanctl --list-sas, unequivocally proving that the captured secure traffic belonged to our newly negotiated IKEv2 tunnel.





