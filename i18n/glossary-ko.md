# Korean glossary

The terms the Korean translation of Nabi follows: the site (nabi-lang/www), the docs (`ko/`) and the spec. Each term is the one most `ko/` pages already use.

When a term is missing, take the one most pages use, add it here in the same change, and keep using it.

## Rules

- **Nabi** is a name and is never translated or transliterated. Lowercase `nabi` is the command and stays in code font.
- Keywords, identifiers, types, commands, flags, file names and paths stay in English and in code font. Do not translate a word in code font.
- Comments and output inside code blocks stay in English, as the tools print them.
- Plain declarative prose (합니다체). Translate the meaning, not the English word order.
- A few terms of art stay in English: panic, feature, toolchain, issue, pull request. A particle follows the English word as it is read: `panic이`, `feature를`, `toolchain을`.

## Terms

| English | Korean | Notes |
|---|---|---|
| trait | 트레잇 | not 트레이트; the keyword is `trait` |
| directory | 디렉터리 | not 디렉토리 |
| iterator | 반복자 | not 이터레이터 |
| parameter | 매개변수 | not 파라미터 |
| argument | 인자 | |
| panic | panic | not 패닉; "it panics": panic이 납니다, "makes it panic": panic을 일으킵니다 |
| toolchain | toolchain | kept in English on most pages; not 툴체인 |
| publish | 공개 | `nabi publish`: 공개합니다; not 게시, 배포 |
| error | 에러 | not 오류 |
| feature (package) | feature | `features` in `nabi.nabion` |
| registry | 레지스트리 | packages.nabi-lang.org |
| dependency | 의존성 | |
| repository | 저장소 | |
| release | 릴리스 | not 릴리즈 |
| target | 타깃 | not 타겟 |
| method | 메서드 | not 메소드 |
| variant | 변형 | |
| thread | 스레드 | |
| example | 예제 | |
| spec (the language spec) | 사양 | |
| zero-width joiner | 폭 없는 결합자(joiner) | U+200C, U+200D |
| related codes (error pages) | 관련 코드: | the last line of an error page |
