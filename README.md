# Quantum-Proof VPN

Build and run a [strongSwan](https://www.strongswan.org) 6.0beta Post-Quantum IKEv2 Daemon with full Post-Quantum Cryptography (PQC) support.  
This prototype implementation is based on the following IETF documents:

- [RFC 9242](https://tools.ietf.org/html/rfc9242): Intermediate Exchange in the IKEv2 Protocol  
- [RFC 9370](https://tools.ietf.org/html/rfc9370): Multiple Key Exchanges in IKEv2  

---

## Table of Contents

1. [Environment Setup](#environment-setup)  
2. [strongSwan Configuration](#strongswan-configuration)  
3. [Starting the IKEv2 Daemons](#starting-the-ikev2-daemons)  
4. [Establishing the IKE SA and First Child SA](#establishing-the-ike-sa-and-first-child-sa)  
5. [Using the IPsec Tunnels](#using-the-ipsec-tunnels)  
6. [Rekeying of CHILD and IKE SAs](#rekeying-of-child-and-ike-sas)  
7. [Supported PQC Algorithms](#supported-pqc-algorithms)  
8. [Notes](#notes)  

---
## 1. Environment Setup <a name="environment-setup"></a>

### Download and Build Dependencies

Manually download and build the required dependencies to run strongSwan with PQC:

### 1. liboqs (Open Quantum Safe Library)

```bash
git clone --branch main https://github.com/open-quantum-safe/liboqs.git
cd liboqs
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/local -DBUILD_SHARED_LIBS=ON ..
make -j$(nproc)
sudo make install 
```
This library provides the PQC algorithms used by strongSwan.

### 2. strongSwan 6.0.5 Beta (with oqs plugin)
```bash
git clone --branch 6.0.5 https://github.com/strongswan/strongswan.git
cd strongswan
./configure --prefix=/usr --sysconfdir=/etc --enable-openssl --enable-pki --enable-swanctl --enable-vici --enable-oqs --enable-frodo
make -j$(nproc)
sudo make install
```
>Make sure OpenSSL supports `oqs-provider` for full PQC support.


## 2. strongSwan Configuration  <a name="strongswan-configuration"></a>
Configure `/etc/strongswan.conf` with options that set startup scripts, logging, and packet fragmentation for large PQC keys:

```bash
charon {
   start-scripts {
      creds = swanctl --load-creds
      conns = swanctl --load-conns
      pools = swanctl --load-pools
   }
   filelog {
      stderr {
         default = 1
      }
   }
   send_vendor_id = yes
   prefer_configured_proposals = no
   fragment_size = 1480
   max_packet = 30000
}
```

 - Loads credentials, connections, and IP pools automatically on
   startup.
  - Logs debug output to standard error.
   
 - Sets fragmentation parameters to accommodate large post-quantum key
   exchanges.

### Load PQC Signature Algorithms in `pki` Tool
```bash
pki {
   load = plugins: random drbg x509 pubkey pkcs1 pkcs8 pkcs12 pem openssl oqs
}
```
This loads the `oqs` plugin supporting lattice-based post-quantum signature algorithms.

## 3. Starting the IKEv2 Daemons  <a name="starting-the-ikev2-daemons"></a>
### On VPN Gateway "moon"
Open a shell in the gateway environment and start the strongSwan daemon (charon), then load configuration:
```bash
docker exec -ti moon /bin/bash
./charon &
swanctl --load-creds
swanctl --load-conns
swanctl --load-pools
```
### On VPN Client "carol"
Similarly, start the client strongSwan daemon and load configuration:
```bash
docker exec -ti carol /bin/bash
./charon &
swanctl --load-creds
swanctl --load-conns
```
## 4. Establishing the IKE SA and First Child SA  <a name="establishing-the-ike-sa-and-first-child-sa"></a>
### Initiate IKEv2 Security Association and First Child SA
Run the following on the client:

```bash
swanctl --initiate --child net
```
- Starts negotiation between client and gateway.
- Uses hybrid key exchange: classical plus multiple PQC algorithms (Kyber, BIKE, HQC).
- Exchanges keys in phases using `IKE_INTERMEDIATE` messages.
- Authenticates peers with Dilithium5 and Falcon1024 certificates.

### Establish a Second CHILD SA
To add another IPsec tunnel:

```bash
swanctl --initiate --child host
```
- Sets up a second tunnel with different crypto proposals (e.g., FrodoKEM).
- Demonstrates simultaneous multi-algorithm key exchanges.

## 5. Using the IPsec Tunnels  <a name="using-the-ipsec-tunnels"></a>
### Test connectivity behind the VPN
Ping the subnet behind the gateway and the gateway’s external IP:

```bash
ping -c 2 10.1.0.2
ping -c 1 192.168.0.2
```
View active Security Associations
```bash
swanctl --list-sas
```
## 6. Rekeying of CHILD and IKE SAs  <a name="rekeying-of-child-and-ike-sas"></a>
- CHILD Security Associations are automatically rekeyed every 20 minutes.
- IKE Security Association is automatically rekeyed every 30 minutes.
- Rekeying uses the same multi-step hybrid key exchange process, including fragmentation for large keys.
## 7. Supported PQC Algorithms  <a name="supported-pqc-algorithms"></a>
### Key Encapsulation Mechanisms (KEM)
- Kyber, BIKE, HQC, FrodoKEM (various security levels)

### Digital Signatures
- Dilithium2/3/5, Falcon512/1024

These algorithms are supported via the `liboqs` library and `strongSwan’s oqs` and `frodo` plugins.

## 8. Notes  <a name="notes"></a>
- This is a research prototype demonstrating PQC integration into IKEv2/IPsec using strongSwan.
- Hybrid mode combines classical and post-quantum cryptographic primitives to maximize security.
- Fragmentation support is essential due to large key and certificate sizes in PQC.

**Author**: Veer Patwa
**License**: [Creative Commons CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
