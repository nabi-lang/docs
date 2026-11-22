# 19. 密码学与 TLS

Nabi 自带密码学实现。`core::crypto` 提供各种算法（哈希、HMAC 与 HKDF、认证加密、密钥交换、签名），`std::random` 读取操作系统的安全随机源，`std::net::tls` 是在这两者之上构建的 TLS 1.3 客户端和服务器。不涉及 OpenSSL 或其他系统库，所以同样的代码在每个目标上行为一致。

> [!WARNING]
> 这些代码是实验性的，尚未经过审计。它用公开的测试向量做了测试，并按照对秘密数据以常数时间运行的方式编写，但还没有经过密码学代码在被信任处理真实秘密之前所需要的独立审查。可以用它来学习和实验；目前凡是重要的场合，请继续使用经过审计的库。

## 哈希与 MAC

常用的项在 `core::crypto` 中（`HashAlgorithm`、`Digest`、`Hmac`、`Aead`、`Secret` 等）。密钥和 HKDF 要通过各自算法的模块访问，因为其中好几个都有 `PublicKey`：`crypto::ed25519::SigningKey`、`crypto::x25519::PrivateKey`、`crypto::hkdf::extract`。这些模块是 `sha2`、`sha3`、`hmac`、`hkdf`、`aead`、`x25519`、`ed25519`、`ecdsa`、`rsa` 和 `mlkem`。每个接受字节的函数都接受任意 `AsBytes` 值：`List<u8>`、`Slice<u8>`、字节数组、`str`（取其 UTF-8 字节）或 `Secret`，所以字节串和文本都可以直接传入。你自己的函数如果要把字节继续传下去，也同样接受 `<B: AsBytes>`；你自己实现了 `AsBytes` 的类型同样可以传入。结果是 `List<u8>`，但密钥材料除外，它以 `Secret` 返回（见下文）。要把字节显示为文本，`encoding::hex::encode` 和 `encoding::base64::encode` 把它们转成十六进制或 base64（`base64::encode(x, base64::URL_SAFE_NO_PAD)` 选用另一种字母表和填充方式），`decode` 再把文本转回来。

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

`Hmac::verify` 以常数时间比较。要自己比较两个秘密，请用 `crypto::constant_time_eq` 而不是 `==`，因为 `==` 在第一个不同处就停止，从而泄露了有多少部分相同。`hkdf::extract` / `expand` 从共享秘密派生密钥。`HashAlgorithm` 也包含 SHA-3 函数 `Sha3_256` 和 `Sha3_512`，所以 HMAC、HKDF 和签名可以像使用其他算法一样使用它们：`Hmac::mac(HashAlgorithm::Sha3_256, key, data)`。`sha3` 还把 SHA-3 和 SHAKE 的状态作为独立类型提供。以后的版本可能会增加算法，所以对 `HashAlgorithm` 的 `match` 需要一个 `_` 分支。

## 加密与签名

`Aead` 一步完成加密和认证（AES-128-GCM、AES-256-GCM 或 ChaCha20-Poly1305）。`seal` 返回附带 16 字节标签的密文；只要密文、标签或关联数据中有任何一项被改动，`open` 就返回 `none`。密钥和 nonce 来自 `random`：算法把随机性当作参数接收，自己从不获取。

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

`x25519` 和 `mlkem`（ML-KEM-768）用于协商共享密钥（分别是经典方式和后量子方式），`ecdsa`（P-256 / P-384）和 `rsa` 为 Web 所用的证书签名和验证。私钥是带类型的值（`ed25519::SigningKey`、`x25519::PrivateKey`、`ecdsa::SigningKey`、`rsa::PrivateKey`、`mlkem::DecapsulationKey768`），在创建时检查一次，其字节保存在秘密存储中（见下文）；它的公开部分（`VerifyingKey`、`PublicKey`、`EncapsulationKey768`）是普通的值。

## 密钥材料：`Secret`

`Secret` 把字节保存在*秘密存储*中：这是一块垃圾回收器从不移动或复制、并在释放前用零覆盖的缓冲区。`hkdf::extract` / `expand`、`x25519::PrivateKey::diffie_hellman` 以及 ML-KEM 的 `encapsulate` / `decapsulate` 都返回 `Secret`，`random::secret(n)` 则把随机字节直接写入一个 `Secret`。它打印为 `Secret { len: n, .. }`，从不显示内容，也没有 `==`：请用 `constant_time_eq` 比较。`wipe()` 立即将它清零，`using` 绑定会在块结束时清除它。`Aead` 和各私钥类型也把密钥保存在秘密存储中，它们的 `wipe` 会清除密钥。

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

把秘密的字节复制到普通对象中（`as_bytes().to_list()`、`str`、格式化字符串）会产生一份垃圾回收器不会清除的副本。秘密存储也没有锁定在内存中，所以仍可能进入交换区或核心转储。

## TLS：`net::tls`

`tls::Client::new(stream, server_name)` 在一个已打开的 `TcpStream` 上运行 TLS 1.3 握手，用操作系统的信任库和服务器名检查服务器的证书链，并返回一个连接，它的 `read` / `write_all` / `read_to_end` 方法与套接字相同。使用默认配置时，握手会提供 X25519MLKEM768 后量子混合方案。TLS 1.2 及更早的版本会被拒绝。

```nabi norun
use std::net::{tcp_connect, tls};

let conn = try tls::Client::new(try tcp_connect("example.com:443"), "example.com");
try conn.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n");
let resp = try conn.read_to_end();
try conn.close();                                           // close_notify, then the socket
println((try str::from_utf8(resp)).lines().next() ?? "");   // HTTP/1.1 200 OK
```

实际的 HTTP 请使用 [`http` 包](18-features-and-publishing.html#nabi-官方包)，它替你完成这些工作，还支持重定向、连接池和 HTTP/2。

服务器需要证书链及其私钥，通常从 PEM 文件中读取。下面的程序在回环接口上运行两端：一个使用自签名证书的服务器任务，以及一个通过 `ClientConfig.root_certs` 只信任该证书的客户端。`ClientConfig` 和 `ServerConfig` 是字段带默认值的记录，所以字面量只需写出不同的部分：`ClientConfig { root_certs: chain }`，或者用 `ClientConfig { alpn_protocols: [b"h2"], ..try tls::ClientConfig::new() }` 从操作系统的信任库开始。每个调用都有用于任务的 `_async` 形式（[第 12 章](12-concurrency.html)）。

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

`tls::system_roots()` 读取操作系统的信任库（`SSL_CERT_FILE` 可以覆盖它），`ClientConfig.alpn_protocols` 协商 `h2` 这样的协议，`negotiated_alpn()` / `peer_certificates()` 告诉你协商的结果。支持客户端证书、带 0-RTT 数据的会话恢复以及 OCSP stapling；不使用 CRL，也不通过网络进行 OCSP 查询。

## 小结

- `core::crypto` 提供 SHA-2 / SHA-3、HMAC、HKDF、AEAD、X25519、ML-KEM、Ed25519、ECDSA 和 RSA。字节参数接受任意 `AsBytes` 值；私钥是保存在秘密存储中的带类型值。
- 随机性只来自 `std::random`（`bytes(n)`、`fill(buf)`、`secret(n)`），并以参数传入。
- 密钥以 `Secret` 返回，可以通过 `wipe()`、`using` 清除，或在被回收时清除。
- 用 `constant_time_eq` 或 `Hmac::verify` 比较秘密，绝不要用 `==`。
- `tls::Client` / `tls::Server` 用 TLS 1.3 包装 `TcpStream`，之后像它一样读写；`close()` 发送 close_notify 并关闭套接字。
- `ClientConfig` / `ServerConfig` 字面量只写出与默认值不同的字段。
- 这份实现是新的，尚未经过审计。

## 延伸阅读

- 标准库规范：[2.21 `crypto`](../spec/refs-standard-libraries/200-core/221-crypto.html)、[3.9 `random`](../spec/refs-standard-libraries/300-std/309-random.html)、[3.2.5 TLS 1.3](../spec/refs-standard-libraries/300-std/302-net.html#325-tls-13---nettls)
- API：[core::crypto](../../std/core/crypto.html)、[std::net::tls](../../std/std/net/tls.html)
- [示例](../examples/index.html)
- 下一章：[20. 怎么做……](20-cookbook.html)
