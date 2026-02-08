# OpenStack 설치 가이드

---

현재 구성된 OpenStack은 **Ubuntu 환경**에 설치되었기에, 설치 명령어들은 모두 **Ubuntu 기반**으로 정리되었습니다.

- **패키지 설치 등 시스템 작업**은 `root` 권한으로 진행
- **유저, 룰, 서비스, 엔드포인트 생성**은 **일반 사용자**로 설정

> NOTE
> endpoint 와 conf 파일의 url 설정에서 hostname으로 설정했을 때 API 간 통신 문제가 발생하는 경우가 있어 적절히 섞어서 사용할 필요가 있음
>
> 아래 설치 가이드에 **controller 또는 mncsvrt06등은 본인의 hostname으로 변경 필요함**
