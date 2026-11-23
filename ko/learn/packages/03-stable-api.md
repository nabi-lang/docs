# 3. API를 안정적으로 유지하기

공개한 패키지의 사용자는 `^0.3` 같은 버전 요구 사항으로 그 패키지에 맞춰 빌드합니다. 그러니 그 범위 안의 릴리스가 사용자의 코드를 깨뜨려서는 안 됩니다. 이 페이지는 API를 없애는 방법과 자랄 여지를 남기는 방법을 보여 줍니다.

## API 없애기

패키지의 함수, 메서드, 필드, 타입을 없애고 싶다면 바로 지우지 말고 `#[deprecated]`를 붙이세요. 그것을 쓰는 코드는 계속 빌드되고, 대신 무엇을 쓰면 되는지 알려 주는 경고를 받습니다. 실제로 지우는 것은 다음 호환성이 깨지는 릴리스에서 합니다.

```nabi norun
#[deprecated(since = "0.3.0", note = "use `wrap_width`")]
pub fn wrap(s: str) -> str { wrap_width(s, 80) }
```

```text
warning[E0131]: use of deprecated function `wrap`
  --> src/main.nabi:6:9
   |
 6 | println(textwrap::wrap("hi"));
   |         ^^^^^^^^^^^^^^^^^^^^
   |
   = note: deprecated since 0.3.0
   = note: use `wrap_width`
   = note: `#[warn(deprecated)]` on by default
```

## 자랄 여지 남기기

자랄 여지를 남기는 속성이 둘 있습니다. enum에 `#[non_exhaustive]`를 붙이면 다른 패키지의 `match`는 `_ =>`로 끝나야 하므로 나중에 변형을 더할 수 있습니다. struct에 붙이면 필드를 더할 수 있는 레코드가 됩니다. 다른 패키지는 여전히 리터럴로 값을 만들고 기본값이 있는 필드는 생략할 수 있지만, 그 struct에 대한 패턴은 `..`로 끝나야 합니다. 나중에 더하는 필드에는 모두 기본값을 주어야 그런 리터럴이 계속 컴파일됩니다.

```nabi norun
#[non_exhaustive]
pub struct WrapOptions {
    pub width: i32 = 80;
    pub break_words: bool = false;
}

// in another package
let o = textwrap::WrapOptions { width: 60 };
let textwrap::WrapOptions { width, .. } = o;
```

트레잇에 `#[sealed]`를 붙이면 그 트레잇은 내 패키지만 구현합니다. 다른 패키지도 바운드에 쓰고 메서드를 부를 수 있지만, 다른 곳의 `impl`은 에러입니다. 그래서 구현이 무엇무엇인지 다 안다고 믿고 코드를 쓸 수 있습니다.

## 더 읽을거리

- 사양: [6.2.7 `#[deprecated]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#627-deprecated-사용-자제-표시), [6.2.10 `#[non_exhaustive]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6210-non_exhaustive-외부-완전성-검사-회피), [6.2.14 `#[sealed]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6214-sealed-정의한-패키지에서만-구현)
- [패키지 공개](index.html)로 돌아가기
