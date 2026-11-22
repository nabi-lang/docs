# 密封消息

只用 `core::crypto` 和操作系统的随机源，加密一条消息，使只有接收者能读取它，并对它签名，使接收者知道是谁发送的。

> [!WARNING]
> Nabi 的密码学实现是新的，尚未经过审计。这个程序展示各个部分如何组合；暂时不要用它保护真正的机密。

## 代码

```nabi
// Send a signed, encrypted message: X25519 key agreement, HKDF, ChaCha20-Poly1305 and Ed25519.
use core::crypto::{Aead, AeadAlgorithm, HashAlgorithm, Secret};
use core::crypto::{ed25519, hkdf, x25519};
use std::random;

struct Sealed {
    sender_key: x25519::PublicKey;   // the sender's one-time X25519 public key
    nonce: List<u8>;
    body: List<u8>;                  // ciphertext and tag
    signature: List<u8>;             // Ed25519 over sender_key, nonce and body
}

/// The AEAD key both sides derive from an X25519 shared secret.
fn derive(shared: Secret) -> Secret {
    let prk = hkdf::extract(HashAlgorithm::Sha256, b"sealed-box v1", shared);
    try! hkdf::expand(HashAlgorithm::Sha256, prk, "aead key", 32)
}

fn signed_part(s: Sealed) -> List<u8> {
    s.sender_key.to_bytes().to_list() + s.nonce + s.body
}

fn seal(to: x25519::PublicKey, signer: ed25519::SigningKey, text: str) -> Sealed {
    let eph = try! x25519::PrivateKey::from_bytes(random::secret(32));
    let shared = eph.diffie_hellman(to) ?? panic("bad peer key");
    let nonce = random::bytes(12);
    let body = try! (try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, derive(shared))).seal(nonce, "", text);
    let s = Sealed { sender_key: eph.public_key(), nonce: nonce, body: body, signature: [] };
    s.signature = signer.sign(signed_part(s));
    s
}

fn open(s: Sealed, me: x25519::PrivateKey, from: ed25519::VerifyingKey) -> str | none {
    if !from.verify(signed_part(s), s.signature) { return none; }
    let shared = me.diffie_hellman(s.sender_key)?;
    let aead = try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, derive(shared));
    let plain = (try! aead.open(s.nonce, "", s.body))?;
    try? str::from_utf8(plain)
}

// Bob's encryption key and Alice's signing key.
let bob = try! x25519::PrivateKey::from_bytes(random::secret(32));
let alice = try! ed25519::SigningKey::from_seed(random::secret(32));
let alice_public = alice.verifying_key();

let msg = seal(bob.public_key(), alice, "the key is under the mat");
println(f"sealed: {msg.body.len()} bytes, signature {msg.signature.len()} bytes");
println(f"bob reads: {open(msg, bob, alice_public) ?? "(rejected)"}");

let eve = try! x25519::PrivateKey::from_bytes(random::secret(32));
println(f"eve reads: {open(msg, eve, alice_public) ?? "(rejected)"}");

msg.body[0] ^= 1;
println(f"tampered: {open(msg, bob, alice_public) ?? "(rejected)"}");
```

```output
sealed: 40 bytes, signature 64 bytes
bob reads: the key is under the mat
eve reads: (rejected)
tampered: (rejected)
```

## 讲解

- 每条消息都使用一对新的 X25519 密钥。它的私密部分和 Bob 的公钥得出的共享密钥，与 Bob 用他的私钥和随消息发来的一次性公钥算出的共享密钥相同。密钥是有类型的值：`x25519::PrivateKey` 把它的字节保存在机密存储中，它的 `PublicKey` 部分是放进消息中的普通值（[`x25519`](../../std/core/crypto/x25519.html)，[2.21.3 密钥交换](../spec/refs-standard-libraries/200-core/221-crypto.html#2213-key-exchange)）。
- 原始的共享密钥不直接用作密钥：`hkdf::extract` / `expand` 把它变成 32 字节的 AEAD 密钥，并用一个标签（`b"sealed-box v1"`）把密钥与这个协议绑定（[2.21.1](../spec/refs-standard-libraries/200-core/221-crypto.html#2211-hashes-hmac-hkdf)）。
- ChaCha20-Poly1305 同时进行加密和认证。24 字节的文本变成 40 字节，多出的 16 字节是认证标签。密钥不对或者有字节被改动时，`open` 返回 `none`，所以 Eve 和被篡改的副本都会在任何明文出现之前被拒绝（[`Aead`](../../std/core/crypto/aead.html)，[2.21.2](../spec/refs-standard-libraries/200-core/221-crypto.html#2212-aead)）。
- Alice 用 Ed25519 对一次性公钥、nonce 和密文一起签名，所以谁也无法替换其中任何一个。`open` 先用 Alice 的 `VerifyingKey` 检查签名（[`ed25519`](../../std/core/crypto/ed25519.html)）。
- 私钥由 `random::secret` 生成，它把字节直接放进机密存储；共享密钥和派生密钥也是 `Secret`，所以任何密钥字节都不会落入普通的 `List`。nonce 是公开的，来自 `random::bytes`。算法从不自己获取随机数（[密钥材料](../learn/19-crypto-and-tls.html#密钥材料secret)）。可选的结果用后缀 `?` 和 `??` 取出（[和类型](../learn/05-sum-types.html)）。
- 更多内容见[密码学与 TLS](../learn/19-crypto-and-tls.html)。
