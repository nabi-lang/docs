# 19. 密碼學與 TLS

Nabi 自帶密碼學實作。`core::crypto` 提供各種演算法（雜湊、HMAC 與 HKDF、認證加密、金鑰交換、簽章），`std::random` 讀取作業系統的安全亂數來源，`std::net::tls` 則是建立在這兩者之上的 TLS 1.3 用戶端與伺服器。整個過程不牽涉 OpenSSL 或其他系統函式庫，所以同一份程式碼在每個目標上的行為都相同。

> [!WARNING]
> 這部分程式碼仍屬實驗性質，尚未經過稽核。它已經用公開的測試向量測試過，並且寫成在處理秘密資料時以常數時間執行，但還沒有經過密碼學在你託付真正的秘密之前所需要的獨立審查。目前請用它來學習和實驗，任何重要的用途仍請使用經過稽核的函式庫。

## 雜湊與 MAC

常用的項目位於 `core::crypto`（`HashAlgorithm`、`Digest`、`Hmac`、`Aead`、`Secret` 等）。金鑰與 HKDF 要透過各自演算法的模組取用，因為其中有好幾個都有 `PublicKey`：`crypto::ed25519::SigningKey`、`crypto::x25519::PrivateKey`、`crypto::hkdf::extract`。這些模組是 `sha2`、`sha3`、`hmac`、`hkdf`、`aead`、`x25519`、`ed25519`、`ecdsa`、`rsa` 和 `mlkem`。每個接收位元組的函式都接受任何 `AsBytes` 值：`List<u8>`、`Slice<u8>`、位元組陣列、`str`（取其 UTF-8 位元組）或 `Secret`，所以位元組字串和文字都可以直接傳入。你自己寫的、要把位元組往下傳的函式，同樣以 `<B: AsBytes>` 接收；你自己實作了 `AsBytes` 的型別也可以傳入。結果是 `List<u8>`，金鑰材料除外，它會以 `Secret`（見下文）回傳。要把位元組顯示成文字，`encoding::hex::encode` 和 `encoding::base64::encode` 會把它們轉成十六進位或 base64（`base64::encode(x, base64::URL_SAFE_NO_PAD)` 可以選擇另一組字母表與填補方式），`decode` 則把文字轉回來。

```nabi
use core::crypto::{HashAlgorithm, Digest, Hmac};
use core::encoding::hex;

println(hex::encode(HashAlgorithm::Sha256.digest("abc")));

// The same hash, fed in pieces.
let d = Digest::new(HashAlgorithm::Sha256);
d.update("a");
d.update(b"bc");
println(f"{d.finish() == HashAlgorithm::Sha256.digest("abc")}");

let tag = Hmac::mac(HashAlgorithm::Sha256, "secret key", "message");
println(f"{tag.len()} {Hmac::verify(HashAlgorithm::Sha256, "secret key", "message", tag)}");
println(f"{Hmac::verify(HashAlgorithm::Sha256, "secret key", "massage", tag)}");
```

```output
ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
true
32 true
false
```

`Hmac::verify` 以常數時間比較。要自己比較兩個秘密值時，請用 `crypto::constant_time_eq` 而不是 `==`；`==` 遇到第一個不同處就停止，因此會洩漏有多少部分相符。`hkdf::extract` / `expand` 從共享秘密衍生金鑰。`HashAlgorithm` 也包含 SHA-3 函式 `Sha3_256` 和 `Sha3_512`，所以 HMAC、HKDF 和簽章都能像其他演算法一樣接受它們：`Hmac::mac(HashAlgorithm::Sha3_256, key, data)`。`sha3` 也把 SHA-3 和 SHAKE 的狀態提供為獨立的型別。之後的版本可能會新增演算法，所以對 `HashAlgorithm` 做 `match` 時需要 `_` 分支。

## 加密與簽章

`Aead` 一步完成加密與認證（AES-128-GCM、AES-256-GCM 或 ChaCha20-Poly1305）。`seal` 回傳附帶 16 位元組標籤的密文；只要密文、標籤或關聯資料任何一項被改動，`open` 就回傳 `none`。金鑰和 nonce 來自 `random`：演算法把亂數當作引數接收，從不自行取得。

```nabi
use core::crypto::{Aead, AeadAlgorithm};
use core::crypto::ed25519::SigningKey;
use std::random;

let key = random::secret(32);
let nonce = random::bytes(12);           // never reuse a nonce with the same key
let aead = try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, key);
let sealed = try! aead.seal(nonce, "v1", "meet at noon");
println(f"{sealed.len()} bytes sealed");  // 12 bytes of text + a 16-byte tag

if let plain = try! aead.open(nonce, "v1", sealed) {
    println(try! str::from_utf8(plain));
}
sealed[0] = sealed[0] ^ 1;              // tamper with one bit
println(f"tampered: {(try! aead.open(nonce, "v1", sealed)) is none}");

let signer = try! SigningKey::from_seed(random::secret(32));   // the private key
let public = signer.verifying_key();
let sig = signer.sign("release 1.0");
println(f"{public.verify("release 1.0", sig)} {public.verify("release 1.1", sig)}");
```

```output
28 bytes sealed
meet at noon
tampered: true
true false
```

`x25519` 和 `mlkem`（ML-KEM-768）用來協商共享金鑰（分別是傳統方式與後量子方式），`ecdsa`（P-256 / P-384）和 `rsa` 則為網路上使用的憑證進行簽署與驗證。私密金鑰是有型別的值（`ed25519::SigningKey`、`x25519::PrivateKey`、`ecdsa::SigningKey`、`rsa::PrivateKey`、`mlkem::DecapsulationKey768`），在建立時檢查一次，並把它的位元組保存在秘密儲存區（見下文）；它的公開部分（`VerifyingKey`、`PublicKey`、`EncapsulationKey768`）則是一般的值。

## 金鑰材料：`Secret`

`Secret` 把位元組保存在**秘密儲存區**（secret storage）：這是一塊垃圾回收器從不搬移或複製、並且在釋放前會用零覆寫的緩衝區。`hkdf::extract` / `expand`、`x25519::PrivateKey::diffie_hellman` 以及 ML-KEM 的 `encapsulate` / `decapsulate` 都回傳 `Secret`，`random::secret(n)` 則把亂數位元組直接抽取到 `Secret` 中。它印出來是 `Secret { len: n, .. }`，從不顯示內容，也沒有 `==`：請用 `constant_time_eq` 比較。`wipe()` 會立即將它歸零，而 `using` 繫結會在區塊結束時清除它。`Aead` 和各種私密金鑰型別同樣把金鑰保存在秘密儲存區，它們的 `wipe` 會將其清除。

```nabi
use core::crypto::{Aead, AeadAlgorithm, HashAlgorithm, Secret, constant_time_eq};
use core::crypto::{hkdf, x25519};
use std::random;

let a = try! x25519::PrivateKey::from_bytes(random::secret(32));
let b = try! x25519::PrivateKey::from_bytes(random::secret(32));
let ab = a.diffie_hellman(b.public_key()) ?? panic("bad key");
let ba = b.diffie_hellman(a.public_key()) ?? panic("bad key");
println(f"{ab:?} {constant_time_eq(ab, ba)}");

let key = try! hkdf::expand(HashAlgorithm::Sha256, hkdf::extract(HashAlgorithm::Sha256, "salt", ab), "key", 32);
let aead = try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, key);
key.wipe();                               // zero it now, not at some later collection
println(f"{key[0]}");
aead.wipe();                              // the cipher is done too

using s = Secret::new("temporary") {
    println(f"{s.len()} bytes");
}                                         // wiped here
```

```output
Secret { len: 32, .. } true
0
9 bytes
```

把秘密值的位元組複製到一般物件（`as_bytes().to_list()`、`str`、格式化後的字串）會產生一份垃圾回收器不會清除的副本。秘密儲存區也沒有鎖定在 RAM 中，所以它仍可能被寫入置換空間或核心傾印檔。

## TLS：`net::tls`

`tls::Client::new(stream, server_name)` 在已開啟的 `TcpStream` 上執行 TLS 1.3 交握，依照作業系統的信任儲存區檢查伺服器的憑證鏈及其名稱，並回傳一個連線，具有與 socket 相同的 `read` / `write_all` / `read_to_end` 方法。使用預設設定時，交握會提供 X25519MLKEM768 後量子混合方案。TLS 1.2 及更舊的版本會被拒絕。

```nabi norun
use std::net::{tcp_connect, tls};

let conn = try tls::Client::new(try tcp_connect("example.com:443"), "example.com");
try conn.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n");
let resp = try conn.read_to_end();
try conn.close();                                           // close_notify, then the socket
println((try str::from_utf8(resp)).lines().next() ?? "");   // HTTP/1.1 200 OK
```

實際的 HTTP 請改用 [`http` 套件](18-features-and-publishing.html#nabi-套件)，它會替你處理這些事，還支援重新導向、連線池和 HTTP/2。

伺服器需要一條憑證鏈及其私密金鑰，通常從 PEM 檔案讀取。下面的程式在 loopback 介面上同時執行兩端：一個使用自簽憑證的伺服器任務，以及一個透過 `ClientConfig.root_certs` 只信任該憑證的用戶端。`ClientConfig` 和 `ServerConfig` 是欄位都有預設值的紀錄，所以字面值只需寫出不同的部分：`ClientConfig { root_certs: chain }`，或是 `ClientConfig { alpn_protocols: [b"h2"], ..try tls::ClientConfig::new() }` 以作業系統的信任儲存區為起點。每個呼叫都有供任務使用的 `_async` 形式（[第 12 章](12-concurrency.html)）。

```nabi
use std::net;
use std::net::{TcpStream, tls};

// A self-signed Ed25519 certificate for `localhost` and its key (a demo
// key: it is public, never use it for anything real).
const CERT: str = "
-----BEGIN CERTIFICATE-----
MIIBVTCCAQegAwIBAgIUOw8IA2ogBBGsxE/m7hJPQIZPLzYwBQYDK2VwMBQxEjAQ
BgNVBAMMCWxvY2FsaG9zdDAgFw0yNjEwMTYxNjEyMDRaGA8yMTI2MDkyMjE2MTIw
NFowFDESMBAGA1UEAwwJbG9jYWxob3N0MCowBQYDK2VwAyEAguLEN9y6fnek02iG
f4LullsxttwNkvHTYdhsjnFWsVajaTBnMB0GA1UdDgQWBBRUjvYk1Hhem6n+Hr1q
HZm6wzG3mzAfBgNVHSMEGDAWgBRUjvYk1Hhem6n+Hr1qHZm6wzG3mzAPBgNVHRMB
Af8EBTADAQH/MBQGA1UdEQQNMAuCCWxvY2FsaG9zdDAFBgMrZXADQQC2XwFaBIQ5
C138MCqnpjXays5b9C4QnQ+tBky5VqKY1ffbKN9O4IIWBeQqED+shdRM5Y4HGYcH
+Zr3PLdVn7QD
-----END CERTIFICATE-----
";
const KEY: str = "
-----BEGIN PRIVATE KEY-----
MC4CAQAwBQYDK2VwBCIEIMa20cYLozv3/41l6/owqHiube5cG+rd7cF4nHPZ6qd3
-----END PRIVATE KEY-----
";

let chain = try! tls::Certificate::from_pem(CERT);
let scfg = try! tls::ServerConfig::new(chain, try! tls::PrivateKey::from_pem(KEY));
let l = try! net::tcp_listen("127.0.0.1:0");
let addr = try! l.local_addr();
let server = task::spawn_local(async {
    let (conn, _) = try! await l.accept_async();
    try! l.close();
    let s = try! await tls::Server::new_async(conn, scfg);
    let buf = List::filled(64, 0u8);
    let n = try! await s.read_async(buf);
    try! await s.write_all_async(b"pong");
    try! await s.close_notify_async();
    let sni = s.server_name() ?? "-";
    try! s.close();
    f"server read {n} bytes, SNI {sni}"
});

let ccfg = tls::ClientConfig { root_certs: chain };   // trust only our certificate
let tcp = try! await TcpStream::connect_async(addr);
let c = try! await tls::Client::with_config_async(tcp, "localhost", ccfg);
try! await c.write_all_async(b"ping");
let reply = try! await c.read_to_end_async();
try! c.close();
println(try! str::from_utf8(reply));
println(await server);
```

```output
pong
server read 4 bytes, SNI localhost
```

`tls::system_roots()` 讀取作業系統的信任儲存區（`SSL_CERT_FILE` 可以覆寫它），`ClientConfig.alpn_protocols` 協商 `h2` 等協定，`negotiated_alpn()` / `peer_certificates()` 則告訴你雙方協議的結果。用戶端憑證、帶 0-RTT 資料的工作階段恢復以及 OCSP stapling 都有支援；CRL 和透過網路的 OCSP 查詢則不使用。

## 小結

- `core::crypto` 提供 SHA-2 / SHA-3、HMAC、HKDF、AEAD、X25519、ML-KEM、Ed25519、ECDSA 和 RSA。位元組引數接受任何 `AsBytes` 值；私密金鑰是保存在秘密儲存區中的有型別值。
- 亂數只來自 `std::random`（`bytes(n)`、`fill(buf)`、`secret(n)`），並以引數傳入。
- 金鑰以 `Secret` 回傳，會被 `wipe()`、`using` 或在被回收時清除。
- 用 `constant_time_eq` 或 `Hmac::verify` 比較秘密值，絕不用 `==`。
- `tls::Client` / `tls::Server` 以 TLS 1.3 包裝 `TcpStream`，之後就像它一樣讀寫；`close()` 會送出 close_notify 並關閉 socket。
- `ClientConfig` / `ServerConfig` 字面值只寫出與預設值不同的欄位。
- 這份實作是新的，尚未經過稽核。

## 延伸閱讀

- 標準函式庫規格：[2.21 `crypto`](../spec/refs-standard-libraries/200-core/221-crypto.html)、[3.9 `random`](../spec/refs-standard-libraries/300-std/309-random.html)、[3.2.5 TLS 1.3](../spec/refs-standard-libraries/300-std/302-net.html#325-tls-13---nettls)
- API：[core::crypto](../../std/core/crypto.html)、[std::net::tls](../../std/std/net/tls.html)
- [範例](../examples/index.html)
- 下一章：[20. 怎麼做……](20-cookbook.html)
