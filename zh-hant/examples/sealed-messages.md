# 密封訊息

只用 `core::crypto` 和作業系統的亂數來源，把訊息加密成只有收件人能讀，並加上簽章讓收件人知道是誰寄的。

> [!WARNING]
> Nabi 的密碼學實作還很新，也尚未經過稽核。這個程式示範各個部件如何組合；目前請不要用它保護真正的機密。

## 程式碼

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

## 說明

- 每則訊息都使用一組全新的 X25519 金鑰對。它的私密半部加上 Bob 的公開金鑰，得到的共享祕密與 Bob 用自己的私密金鑰和隨訊息送出的一次性公開金鑰算出的相同。金鑰是有型別的值：`x25519::PrivateKey` 把位元組存放在祕密儲存區中，而它的 `PublicKey` 半部是放進訊息裡的一般值（[`x25519`](../../std/core/crypto/x25519.html)、[2.21.3 金鑰交換](../spec/refs-standard-libraries/200-core/221-crypto.html#2213-金鑰交換)）。
- 原始的共享祕密不會直接當作金鑰使用：`hkdf::extract` / `expand` 把它轉成 32 位元組的 AEAD 金鑰，並加上一個標籤（`b"sealed-box v1"`），把金鑰綁定到這個協定（[2.21.1](../spec/refs-standard-libraries/200-core/221-crypto.html#2211-雜湊hmachkdf)）。
- ChaCha20-Poly1305 同時加密與驗證。24 位元組的文字變成 40 位元組，多出的 16 位元組是標籤。金鑰錯誤或有任何位元組被改動時，`open` 會回傳 `none`，所以 Eve 和被竄改的複本在任何明文出現之前就被拒絕（[`Aead`](../../std/core/crypto/aead.html)、[2.21.2](../spec/refs-standard-libraries/200-core/221-crypto.html#2212-aead)）。
- Alice 用 Ed25519 對一次性金鑰、nonce 和密文一起簽章，所以沒有人能替換其中任何一項。`open` 會先用 Alice 的 `VerifyingKey` 驗證簽章（[`ed25519`](../../std/core/crypto/ed25519.html)）。
- 私密金鑰由 `random::secret` 產生，它把位元組直接放進祕密儲存區；共享祕密和衍生出的金鑰也都是 `Secret`，所以沒有任何金鑰位元組落在一般的 `List` 中。nonce 是公開的，來自 `random::bytes`。演算法本身從不自行取得亂數（[金鑰材料](../learn/19-crypto-and-tls.html#金鑰材料secret)）。可選的結果用後綴 `?` 和 `??` 取出（[和型別](../learn/05-sum-types.html)）。
- 更多內容請見[密碼學與 TLS](../learn/19-crypto-and-tls.html)。
