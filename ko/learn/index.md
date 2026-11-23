# Nabi 배우기

Nabi를 처음 접하는 사람을 위한 가이드입니다. 설치부터 에러 처리, 동시성, FFI와 C 소스, NABION, 도구, 패키지, 암호, 쿡북, 배열과 수치 계산, 과학 계산 패키지, 플러그인, 리플렉션, 가비지 컬렉터 일시 정지, 핫 리로드, Objective-C와 Apple 프레임워크, iOS까지 다룹니다. 각 장은 짧은 설명과 바로 실행할 수 있는 코드로 되어 있습니다.

이 가이드의 모든 코드 블록은 사이트를 빌드할 때 실제 toolchain으로 컴파일하고 실행합니다. 실행하지 않는다고 표시한 몇몇 블록(코드 조각, REPL 세션, 끝나지 않는 프로그램)은 보여 주기만 합니다. 프로그램 뒤에 출력 블록이 있으면 실제 출력이 그 출력과 같은지 검사합니다. 아무 블록이나 복사해 `nabi run file.nabi`로 실행해 보세요.

Python을 써 왔다면 [Python 사용자를 위한 Nabi](nabi-for-python.html)에서 Python 관용구와 표준 라이브러리 모듈이 Nabi의 무엇에 해당하는지 볼 수 있습니다.

Rust를 써 왔다면 [Rust 사용자를 위한 Nabi](nabi-for-rust.html)에서 소유권, 에러, `Option`에 대해 버려야 할 습관과 Cargo 명령이 Nabi의 어디에 있는지 볼 수 있습니다.

과학 계산이나 데이터 분석을 한다면 [Nabi로 하는 과학 계산](22-science.html)이 과학 패키지를 정리하고, 각 패키지의 [문서](https://docs.nabi-lang.org/packages)가 튜토리얼로 시작합니다. 처음 해 보는 CSV 분석에서 시작해 선형대수, 통계, 신호, 피팅, 미분방정식, 도함수까지 다룹니다. [Rosetta](rosetta/index.html) 페이지는 FizzBuzz, 소수 체, 여덟 퀸, 단어 세기처럼 잘 알려진 작은 문제를 Nabi로 풉니다.

전체 규칙은 [언어 참조](../spec/refs-lang/index.html)와 [표준 라이브러리 사양](../spec/refs-standard-libraries/index.html)에 있습니다. 함수와 타입은 [표준 라이브러리 API](../../std/index.html)에 정리해 두었습니다. 작고 완결된 프로그램은 [예제](../examples/index.html)에 모아 두었습니다. 파일 읽기, 시간 재기, 스레드, JSON, HTTP 같은 흔한 작업은 [어떻게 하나요](20-cookbook.html)에서 바로 답을 찾을 수 있습니다. 컴파일러가 에러를 내면 그 에러 코드를 [에러 코드 색인](../errors/index.html)에서 찾아보세요.

## 목차
