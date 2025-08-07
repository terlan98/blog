---
title: "🔐 Building iOS apps that can survive the quantum era"
description: "Why now is the time to go quantum-secure"
date: "2025-08-07"
FRtags: ["ios", "swift", "cryptography", "quantum"]
aliases: ["quantum-secure-cryptography"]
ShowToc: true
TocOpen: false
---
> Note: This is a slightly modified copy of my original WWDC25 [note](https://wwdcnotes.com/documentation/wwdcnotes/wwdc25-314-get-ahead-with-quantumsecure-cryptography)

## Key Takeaways

- 🔐 Passive quantum attacks are already relevant
- ⚡ Sensitive data needs quantum-secure encryption immediately
- 🛡️ iOS 26 provides automatic defence mechanisms
- 🆕 New CryptoKit APIs: Post-quantum HPKE (X-Wing), ML-KEM, ML-DSA

## Understanding Quantum Attacks
While today’s encryption is secure against classical computers, quantum computing changes the rules. Even though sufficiently powerful quantum computers are not yet publicly available, attackers have already found ways to exploit the future power of quantum computing.

Types of attacks:
- Passive - does not require a quantum computer at the time of attack
- Active - requires the attacker to have access to a quantum computer

### Passive Attack - Harvest now, decrypt later
<img src="../images/WWDC25-314-HNDL-Attack.jpg#center" width=85%>

**How it works:**
1. Attacker harvests encrypted data by observing the network
2. Attacker waits until sufficiently powerful quantum computers become available
3. Attacker decrypts harvested data using quantum computers

> Important: Since this is a **passive attack**, it is already relevant today.

This attack affects encryption and breaks confidentiality.

### Active Attack
<img src="../images/WWDC25-314-Active-Attack.jpg#center" width=85%>

**How it works:**
1. Attacker intercepts a signature by observing network traffic
2. Attacker steals the signing key from the signature using a quantum computer
3. Attacker uses the stolen key to impersonate the user

> Note: Since we don't have sufficiently powerful quantum computers yet, this is a **future threat**.

This attack affects digital signatures and breaks authenticity.

## Quantum-secure cryptography
New quantum-secure cryptographic algorithms can be adopted today on traditional (non-quantum) devices.

### Public-key cryptography
<img src="../images/WWDC25-314-Public-Key-Cryptography.jpg#center" width=85%>

Current algorithms are based on computationally-intensive mathematical problems, which can be solved by quantum computers. Apple suggests using **Post-quantum Hybrid Public Key Encryption (HPKE)** and **Post-quantum Hybrid Signatures** to secure encryption and digital signatures in your apps. 

Both of these algorithms use a hybrid approach, integrating post-quantum and traditional algorithms. The advantage of this approach is that attacking them demands breaking both layers.

### Symmetric-key cryptography
While these algorithms also rely on complex computational problems, quantum computers reduce their security only by a small constant factor. The solution is to simply double the key size. For example, upgrade 128-bit ciphers to 256-bit.

## Protecting network data
Protecting against Harvest Now, Decrypt Later attacks is the **top priority** now. To achieve this, quantum-secure encryption should be applied to data in transit.

> Note: Apple is already using quantum-secure encryption in iMessage since iOS 17.4

### TLS 1.3
- Includes a quantum-safe encryption enhancement, using a secure key exchange resistant to quantum attacks
- Most providers already support it

### Client-side quantum-secure TLS
Starting from iOS 26, quantum-secure encryption in TLS is enabled by default for:
- URLSession
- Network.framework

> Important: It is recommended to migrate away from legacy APIs like [Secure Transport](https://developer.apple.com/documentation/security/secure-transport), as they won't support quantum-secure TLS

### Server-side quantum-secure TLS
Quantum-secure TLS should also be enabled on the server side.
- Most content/website hosting providers already support and enable it by default
- For your own servers, you need to upgrade TLS explicitly

> Note: iOS 26 introduces support for quantum-secure TLS across various built-in services such as **CloudKit, Apple Push Notifications, iCloud Private Relay, Safari, Weather, Maps**, and other built-in apps

## Protecting custom protocols
In most cases, quantum-secure TLS provides sufficient protection against Harvest Now, Decrypt Later attacks. However, direct usages of cryptographic APIs in your app also require upgrading to their quantum-secure counterparts using **new CryptoKit APIs**. 

### Quantum-secure encryption with CryptoKit
- Based on **Post-quantum HPKE**
- Uses [**X-Wing Key Encapsulation Mechanism (KEM)**](https://eprint.iacr.org/2024/039) that uses [**ML-KEM**](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf) as its post-quantum building block
    - Large encryption size overhead
    - Comparable or even better performance in comparison to traditional methods
    - Hardware-isolated execution via Secure Enclave
    - Formally verified

Sample Post-quantum HPKE usage by Apple:
```swift
let ciphersuite = HPKE.Ciphersuite.XWingMLKEM768X25519_SHA256_AES_GCM_256

// Recipient
let privateKey = try XWingMLKEM768X25519.PrivateKey.generate()
let publicKey = privateKey.publicKey

// Sender
var sender = try HPKE.Sender(recipientKey: publicKey, ciphersuite: ciphersuite, info: info)
let encapsulatedKey = sender.encapsulatedKey

// Recipient
var recipient = try HPKE.Recipient(privateKey: privateKey, ciphersuite: ciphersuite, info: info, encapsulatedKey: encapsulatedKey) 

// Sender encrypts data
let ciphertext = try sender.seal(userData, authenticating: metadata)

// Recipient decrypts message
let decryptedData = try recipient.open(ciphertext, authenticating: metadata)
#expect(userData == decryptedData)
```

While  post-quantum HPKE provides a simplified abstraction, CryptoKit also offers access to **lower-level APIs** for developers.

### Quantum-secure signatures with CryptoKit
Post-quantum hybrid signatures can be made using [**ML-DSA**](https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.204.pdf)
- Also offers hardware-isolated execution via Secure Enclave

Check out [Apple's demo project](https://developer.apple.com/documentation/CryptoKit/using-the-quantum-secure-apis) for sample usages

<img src="../images/WWDC25-314-Post-Quantum-CryptoKit.jpg#center" width=35%>


> Note: CryptoKit’s adherence to standard protocols means it can interoperate with any compliant server library, but Apple's preferred choice is **Swift Crypto**.


## What to do next
- [ ] Replace custom encryption and signatures with their post-quantum counterparts
- [ ] Use URLSession or Network.framework for networking
- [ ] Update server configuration to support quantum-secure TLS

The best time to go quantum-safe was yesterday. The second best time is today.