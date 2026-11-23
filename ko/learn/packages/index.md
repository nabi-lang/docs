# 패키지 공개

내 라이브러리나 프로그램을 패키지 레지스트리 [packages.nabi-lang.org](https://packages.nabi-lang.org)에 올리고 관리하는 방법을 다룹니다. 패키지를 쓰는 법은 [11장](../11-modules-packages.html#패키지-레지스트리)에, feature는 [18장](../18-features-and-publishing.html)에 있습니다.

> [!IMPORTANT]
> 지금은 레지스트리 관리자만 패키지를 공개할 수 있습니다. 누구나 로그인해 패키지를 쓸 수 있지만, `nabi login`은 아직 공개 권한이 없는 계정을 거절합니다.

## 레지스트리의 동작 방식

레지스트리의 데이터는 git 저장소 하나, [index](https://github.com/nabi-lang/packages-index)입니다. 패키지의 버전 하나가 그 안의 줄 하나이고, 그 줄은 패키지 자신의 git 저장소에 있는 커밋 하나와 그 커밋의 트리 해시를 가리킵니다. 소스는 올리지 않습니다. 버전이 공개되면 레지스트리가 그 커밋을 가져와 아카이브로 보관하고, 아카이브의 SHA-256을 줄에 적습니다. 사용자는 아카이브를 받아 index와 대조하므로, 버전은 저장소가 없어져도 남습니다. 줄은 레지스트리만 씁니다. index는 pull request를 받지 않습니다. 줄의 형식은 [index README](https://github.com/nabi-lang/packages-index#readme)에 있습니다.

nabi-lang 조직에서 공개한 패키지에는 **Official** 표시가 붙습니다. 이 패키지들은 toolchain과 함께 옵니다([11장](../11-modules-packages.html#패키지-레지스트리)).

## 페이지

| 페이지 | 다루는 내용 |
|------|----------------|
| [패키지 공개하기](01-publishing.html) | 패키지에 필요한 것, `nabi login`, `nabi publish`, 이름과 소유자, 레지스트리가 만드는 문서, `nabi yank` |
| [신뢰할 수 있는 공개](02-trusted-publishing.html) | 저장해 둔 토큰 없이 GitHub Actions에서 공개하기 |
| [API를 안정적으로 유지하기](03-stable-api.html) | `#[deprecated]`, `#[non_exhaustive]`, `#[sealed]` |
| [보안 권고](04-security-advisories.html) | `nabi audit`, 권고 읽기, 취약점 제보하기 |

레지스트리 자체의 규칙은 레지스트리에 있습니다. [이용 정책](https://packages.nabi-lang.org/policy), [이용 약관](https://packages.nabi-lang.org/terms), [개인정보 처리방침](https://packages.nabi-lang.org/privacy)입니다.
