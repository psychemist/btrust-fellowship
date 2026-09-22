# Building a Sphinx Onion Packet in Python: Part 1

In this post series, we will be implementing the Lightning Sphinx packet in Python.
Before reading this, I encourage you to read two materials, where the author does a fantastic job of setting up the design decisions and requirements for this exercise, far more than I could if I tried.  
The [first](https://www.ellemouton.com/posts/onion-routing-prelims/) establishes the payment route and information that flows through each node. The [second](https://www.ellemouton.com/posts/sphinx/) details the cryptography and construction of the Sphinx packet.  

This article turns the explanations into Python code that works correctly when tested against the official BOLT 04 test [vector](https://github.com/lightning/bolts/blob/master/04-onion-routing.md#test-vector).

## Introduction

Throughout the article we will use this basic payment route:

```
Alice ──▶ Bob ──▶ Charlie ──▶ Dave
```

Alice is aware of the public key of every node on the route and the payload each node should receive. Her node turns all that information into a single packet. Bob will partially decrypt it, forward the remaining data to Charlie, and so on, until it reaches Dave. No node should learn more than what it needs to decrypt and pass on.

### What Sphinx Fixes

![ognion-layers](./images/wk-0304/onion-layers.webp)

The onion packet takes its structure from the literal vegetable. You can visualize each payload as the layer of an onion, with the receiver's payload being the deepest layer. Each node from the first hop strips its layer and forwards the rest of the onion. However, this method actually leaks some sensitive information.

1. For one, nodes can read payloads of those after them i.e. Bob can read Charlie's and Dave's payloads.
2. Since the onion packet is shrinking at every hop, each node can estimate the number of hops left to the destination.
3. If Alice signs the entire packet onion with her node's key, all the nodes on the route learn who the real sender is.

Sphinx is an improvement of the onion model as it enforces encryption at each hop. It limits the scope of data each node can read from the routing data and requires each hop to sign data using an ephemeral (temporary) public key.  
Moreover, the routing data is fixed at 1300 bytes and each hop can decrypt only its payload, its sending and receiving nodes, and nothing else. Since each layer of the onion is a fixed size, a node cannot determine if the next node is the last hop or not.

The Version 0 `onion_packet` consists of a 1-byte version field, a 33-byte compressed secp256k1 public key, 1300 bytes of hop payload data, and a 32-byte HMAC, adding up to 1366 bytes in total.

### Cryptographic Pieces

Before building the packet, we will map each cryptographic primitive to a small function that implements it. The [python-secp256k1](https://pypi.org/project/python-secp256k1/) library is used for all elliptic curve functions throughout the article.

#### Elliptic Curve Diffie-Hellman

Instead of using one private key to encrypt data for the entire route, the sender (Alice) will use a different shared secret for each hop. This shared secret is computed from a public key she owns and the public key of each node along the route.  
For example, say she generates an ephemeral private key (e) and multiplies it by Bob's public key (B) to generate another public key (K). Due to ECDH math, she can send the ephemeral public key (E) to Bob, who multiplies it with his private key (b) and generates the same public point (K) she got.  
This public point is serialized (into bytes) and compressed (33 byte x-coordinate with parity prefix) then hashed to get the 32-byte shared secret.

```python
def compute_shared_secret(priv_bytes, pub_bytes):
    """Compute ECDH shared secret :: SHA256(compress(priv * pub))"""
    pubkey = ec_pubkey_parse(pub_bytes)
    return ecdh(priv_bytes, pubkey)
```

This helper function lets the sender convert a private key into the 33-byte compressed public key that goes into the packet:

```python
def priv_to_pub(priv_bytes):
    """Derive 33-byte compressed public key bytes from 32-byte private key"""
    pubkey = ec_pubkey_create(priv_bytes)
    return ec_pubkey_serialize(pubkey, compressed=True)
```

#### Ephemeral Keys

For privacy reasons, Alice cannot send her public key to every node on every hop of every payment route she interacts with. Instead, she gives each hop a different ephemeral key. To save space, she generates a single session key for that route and computes the first ephemeral key for the first hop. Subsequent hops derive their ephemeral keys using a blinding factor.

The blinding factor for a hop is:
```
SHA256(ephemeral_public_key || shared_secret)
```

The sender multiplies her current ephemeral private key by the blindinng factor modulo the secp256k1 curve order. The hop multiplies the ephemeral public key it received by the same factor. Both parties produce two halves of the same key pair and the hop can compute the factor because it knows the shared secret from the previous section. `sha256` is a function from the built-in `hashlib` library.

```python
def calc_blinding_factor(ephemeral_pubkey, shared_secret):
    """Compute blinding factor :: SHA256(ephemeral_pubkey_compressed || shared_secret)"""
    return sha256(ephemeral_pubkey + shared_secret).digest()


def blind_private_key(priv_bytes, blinding_factor):
    """Blind private key :: (privkey * blinding_factor) mod curve order n"""
    return ec_seckey_tweak_mul(priv_bytes, blinding_factor)
```

#### Hash-Based Message Authentication Code

This is a hash function that assures the integrity of a message shared over public channels. Basically, a sender with a shared secret signs over a message and it can later be verified by the receiver using the same secret. Since Alice is the only party that has shared secrets with all the other nodes, she has to include the HMACs for every party within their layer in the onion packet. Intermediate hops can only see the HMAC intended for them.

```python
def calculate_hmac(key, message):
    """Calculate HMAC-SHA256 digest used in onion MAC"""
    return hmac.new(key, message, sha256).digest()
```

BOLT 04 defines these key types: `rho`, `mu`, `um`, `ammag`, and `pad`. 
`rho` is the stream cipher for obfuscating the routing data, `mu` encrypts the HMAC over it, and `pad` encrypts the initial padding. `um` and `ammag` are used for error messages traveling back to the sender, so we do not need them for constructing or peeling the onion.
The HMAC library is used to derive these keys from a shared secret. BOLT 04 uses the key type as the key and the shared secret as the message:

```python
def generate_key(key_type, shared_secret):
    """Compute HMAC-SHA256(key=key_type, msg=secret)"""
    return hmac.new(key_type, shared_secret, sha256).digest()
```

#### ChaCha20 and XOR

Sphinx hides routing data by XORing it with a pseudorandom byte stream. BOLT 04 spec generates the stream by encrypting zero bytes of a specified length under the `rho` key and a 12-byte nonce filled with zeros. Ordinarily, a zero nonce would be cryptographically unsafe. It is usable here because every key is derived from a new shared secret and is used to encrypt only one thing.

XOR (Exclusive-OR) is a bitwise operation that outputs 1 only when both bits are different i.e one bit must be 0 and the other 1. There are many cool properties of XOR outlined in the linked article that make it useful for packet construction, specifically using a random byte stream to encrypt data so only the receiver and sender can read it. Essentially, if you XOR the plaintext with the pseudorandom byte stream, you get the ciphertext. When the ciphertext is XOR'd against the same byte stream, the plaintext is recovered.

I implement both primitives using the [PyCryptodome](https://www.pycryptodome.org/src/introduction) library.

```python
def generate_cipher_stream(key, length):
    """Generate pseudo-random byte stream with ChaCha20"""
    nonce = bytes(12)
    plaintext = bytes(length)
    cipher = ChaCha20.new(key=key, nonce=nonce)
    return cipher.encrypt(plaintext)
```

### Preparation

At this point, Alice has:
- the session key
- associated data
- public keys of Bob, Charlie, and Dave
- payload for each of them

The associated data is really the Lightning payment hash. it is included in each HMAC, tying all the onion layers to a specific Hashed Time-Lock Contract (HTLC). Thus way, thew whole onion is locked to a specific payment route.

Each hop payload has a length prefix. This prefix is encoded as BigSize, a variable-length unsigned integer. BigSize is similar to Bitcoin's CompactSize but is serialized in big-endian byte order as opposed to little-endian. 

```python
payload = bytes.fromhex(hop["payload"]) 
```

For lengths under 253 bytes, it is a single byte denoting the size. Larger payloads start from `0xfFD` and are stored as a 2-byte big-endian integer. Much larger payloads have `0xFE` and `0xFF` prefixes and are stored as 4-byte and 8-byte integers respectively.

Thus, each hop's data frame looks something like this:

```
[BigSize length][payload][32-byte HMAC for next hop]
```

In the next part, we will go over each step of the actual packet construction and then hadle the peeling steps afterward.
