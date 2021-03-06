표준 시간대 처리 방식
---------------------

### 개요

표준 시간대를 하나(UTC+9)밖에 사용하지 않고 일광절약 시간제(서머타임)도 없는 한국에서는
표준 시간대 처리의 필요성을 별로 느끼지 못합니다.
서버도 UTC+9, 대부분의 사용자들도 UTC+9로 설정해 놓으면 모두 같은 시간을 보게 됩니다.

그러나 외국 사용자들에게 표준 시간대 처리는 매우 중요한 문제입니다.
예를 들어 미국은 본토에만 4개의 표준 시간대(EST, CST, MST, PST)가 있고,
알래스카와 하와이도 각각 다른 표준 시간대를 사용합니다.
게다가 대부분의 지역에서 서머타임을 실시하기 때문에 매년 봄, 가을에 시간이 바뀌어서
많은 웹사이트 관리자들의 골치를 썩이곤 합니다.
그래서 미국이나 유럽에서 널리 사용되는 CMS들은 표준 시간대 자동 변환 기능을 포함하고 있는 경우가 많습니다.

최근에는 한국어 사이트들도 저장 공간과 트래픽의 제한이 적은 해외 호스팅을 사용하는 일이 늘어나서
표준 시간대 처리가 더이상 남의 일이 아니게 되었습니다.
해외 호스팅 서버들은 현지의 표준 시간대로 맞추어져 있는 경우가 많고,
현지 규칙에 따라 서머타임을 적용하기 때문입니다.

XE에서 사용하던 표준 시간대 처리 방식은 이런 복잡한 상황에 대응하지 못하기 때문에
라이믹스에서는 새로운 방식을 도입하였습니다.
XE와 최대한 호환되도록 구현했기 때문에 대부분의 서드파티 자료는 그대로 작동하겠지만,
시간과 관련된 기능을 작성하실 때는 아래의 정보를 참고하시기 바랍니다.

### XE의 표준 시간대 처리 방식

XE의 표준 시간대는 관리 모듈에서 설정할 수 있으며, `files/config/db.config.php`에 `time_zone`이라는 설정으로 저장됩니다.
이 때 설정하는 표준 시간대는 게시판 목록 화면, 글읽기 화면 등에서 **화면에 표시되는 시간**을 의미합니다.

문서와 댓글의 작성 일시, 회원의 가입 및 최근 로그인 일시 등을 DB에 저장할 때는 서버의 표준 시간대를 그대로 사용합니다.
화면에 표시할 때 문서와 댓글의 `getRegdate()` 함수에서 `zdate()` 함수를 호출하면, 여기서 다시 `ztime()` 함수를 호출하여
DB에 저장된 서버 기준의 날짜를 화면에 표시할 날짜로 변환해 주게 됩니다.

문제는 DB에 저장된 날짜가 어느 표준 시간대 기준인지, 즉 서버의 표준 시간대가 무엇인지는 기록되지 않는다는 점입니다.
변환하는 시점에서 `date('O')`를 사용하여 `time_zone` 설정과의 차이를 비교하는 것 뿐입니다.
따라서 국내 서버로 운영하던 사이트를 해외 서버로 옮기거나, 해외 서버에서 서머타임을 사용하기 시작하면
예전에 작성된 글의 날짜가 모두 몇 시간씩 달라질 수 있습니다.

예전에 작성된 글의 날짜가 잘못 표시되는 것은 사이트 운영 도중 관리 모듈에서 표준 시간대를 변경한 경우에도 마찬가지입니다.
분쟁이 발생할 경우 중요한 증거가 될 수도 있는 글 작성 시간의 신뢰도가 떨어지는 것입니다.

관리자가 설정한 표준 시간대밖에 사용할 수 없기 때문에, 전세계 사용자들을 대상으로 서비스할 때 불편한 점도 있습니다.
화면에 표시되는 시간을 뉴욕 기준으로 할 수도 있고 런던 기준으로 할 수도 있지만,
어느 한쪽을 선택하더라도 다른 쪽 사용자는 불편할 수밖에 없습니다.
서머타임에도 전혀 대응이 되지 않습니다.

### 라이믹스의 표준 시간대 처리 방식

이런 불편을 해결하기 위해 라이믹스에서는 아래와 같이 3가지 표준 시간대를 각각 별도로 관리합니다.

- 서버 자체의 표준 시간대
- 날짜를 DB에 저장할 때 사용하는 "내부 표준 시간대" (internal timezone)
- 날짜를 화면에 표시할 때 사용하는 "디스플레이 표준 시간대" (display timezone)

#### 서버 자체의 표준 시간대

서버 자체의 표준 시간대는 무시합니다. 어떤 시간대를 사용하든 시간이 틀리지만 않으면 됩니다.

#### 내부 표준 시간대

내부 표준 시간대는 DB에 날짜를 저장할 때는 물론, PHP의 `date()` 함수 등 시간과 관련된 모든 기능을 사용할 때 일관성있게 적용됩니다.
서버를 옮기거나 서머타임을 사용하더라도 이미 저장된 날짜가 달라지는 일이 없도록,
내부 표준 시간대는 라이믹스 신규 설치 또는 XE에서 업그레이드하는 시점에 결정되며
이후 어떤 일이 있어도 바뀌지 않습니다.

내부 표준 시간대는 `locale.internal_timezone` 설정에 저장되며, 관리자가 변경할 수도 없고 변경할 필요도 없습니다.
신규 설치 또는 XE에서 업그레이드할 때 한국어 사이트는 UTC+9 (32400초), 그 밖의 경우에는 UTC (0초)로 고정됩니다.
(한국어 사이트만 별도로 취급하는 이유는 XE에서 대부분 UTC+9로 사용해 왔기 때문에 호환성 유지를 위해서이며,
더이상 호환성 문제가 발생하지 않는다고 판단되면 언어와 무관하게 UTC로 통일할 예정입니다.)

플러그인(모듈, 애드온 등)에서 DB에 날짜를 저장할 때는 내부 표준 시간대를 사용해야 합니다.

`date()`, `time()` 등의 PHP 내장 함수, 그리고 라이믹스에서 제공하는 `getInternalDateTime()` 함수를 사용하여
내부 표준 시간대 기준의 날짜를 얻을 수 있습니다.

시간 저장 형식은 XE의 관례를 따라 `YYYYMMDDHHMMSS`(`YmdHis`)를 주로 사용하지만, 서드파티 자료에서는 유닉스 타임스탬프를 사용해도 무방하며,
라이믹스에서도 추후 표준 시간대 처리 효율을 높이기 위해 유닉스 타임스탬프로 변경할 가능성이 있습니다.
(단, 유닉스 타임스탬프 사용시 2038년 이후의 날짜를 정상적으로 표시하기 위해 반드시 `BIGINT` 자료형으로 테이블을 구성하여야 합니다.)

#### 디스플레이 표준 시간대

디스플레이 표준 시간대는 문서, 댓글 등의 작성 시간을 화면에 표시할 때 적용되는 설정입니다.
XE와 마찬가지로 관리 모듈에서 선택할 수 있으나,
XE와 달리 사이트 운영 도중 변경하더라도 예전에 작성한 글의 날짜가 잘못 표시되지 않습니다.

관리자가 선택한 기본값은 `locale.default_timezone` 설정에 저장되며, 현재는 모든 사용자에게 일괄 적용되지만
추후 회원마다 자신이 사용하는 표준 시간대를 선택할 수 있도록 개선할 예정입니다.
(지금도 `$_SESSION['timezone']`에 `Asia/Seoul`과 같은 표준 시간대 명칭을 저장하면 해당 사용자의 디스플레이 표준 시간대를 변경할 수 있습니다.
설정 화면이 만들어지지 않았을 뿐입니다.)

XE와 마찬가지로 `zdate()` 함수를 사용하여 내부 표준 시간대를 디스플레이 표준 시간대로 변환할 수 있습니다.
서머타임을 사용하는 표준 시간대를 선택하더라도 항상 정확하게 변환됩니다.

테마(레이아웃, 스킨), 위젯 등에서 화면에 날짜를 표시할 때는 디스플레이 표준 시간대를 사용해야 합니다.

`getDisplayDateTime()` 함수를 사용하여 디스플레이 표준 시간대 기준의 날짜를 얻을 수 있으며,
문서와 댓글의 `getRegdate()` 메소드를 사용해도 됩니다.

### 요약 및 정리

- XE에서는 서버 자체의 표준 시간대와 관리자가 지정한 표준 시간대(라이믹스의 "디스플레이 표준 시간대"에 해당)밖에 사용하지 않았기 때문에,
  둘 중 하나가 변경될 경우 날짜가 잘못 표시되는 문제가 있었습니다.
- 라이믹스에서는 변하지 않는 "내부 표준 시간대"를 기준으로 삼기 때문에,
  서버를 옮기거나 "디스플레이 표준 시간대"를 변경하더라도 항상 일관성있는 날짜 표시를 보장합니다.
- XE에서 사용하던 시간 관련 함수들과 PHP 내장 함수들을 모두 그대로 사용할 수 있습니다.
- 서머타임 등 복잡한 것들은 라이믹스에서 자동으로 계산해 주므로, 어떠한 별도 처리도 필요하지 않습니다.
- 서드파티 자료 개발시 회원마다 서로 다른 표준 시간대를 사용할 수도 있음를 반드시 기억해야 합니다.
