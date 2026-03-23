# 2026-03-22 | '숨고' 스피치 컨설팅 견적 요청 플로우 이벤트 QA

> 내부 이벤트 스키마와 수집 파이프라인에 직접 접근할 수 없는 상황에서, **Amplitude Event Explore**를 활용해 실제 브라우저에서 발생한 이벤트를 관찰하고, `Observed Tracking Plan`과 QA 검증 포인트를 정리한 프로젝트입니다.

---

## 1. 왜 이 프로젝트를 했는가

데이터 QA를 제대로 이해하려면 결국 **이벤트가 실제로 어떻게 수집되는지**를 봐야 한다고 생각했습니다.  
하지만 서비스의 내부 스키마 문서, Amplitude 프로젝트 설정, warehouse 테이블에는 접근할 수 없습니다.

그래서 이번 스터디에서는 접근 가능한 관찰 수단인 **Amplitude Event Explore**를 활용해,

- 숨고 플랫폼 내에서 어떤 이벤트가 실제로 발생하는지
- 어떤 파라미터가 함께 붙는지
- 어떤 지점에서 QA 관점의 검증 포인트가 생기는지

를 직접 정리해보는 방식으로 접근했습니다.

이 프로젝트의 초점은 **제한된 관찰 환경에서 로그 QA 관점으로 이벤트 수집 상태를 읽어내는 과정**에 있습니다.

---

## 2. 관찰 대상 플로우

이번에 관찰한 흐름은 숨고의 **취업 준비 프리셋 진입 후, 스피치 컨설팅 견적 요청을 진행하는 사용자 여정**입니다.

관찰 범위는 아래와 같습니다.

1. 메인 홈 진입
2. 취업 준비 프리셋 진입
3. 스피치 컨설팅 선택 및 요청폼 진행
4. 게스트 사용자의 로그인 게이트 노출
5. 제출 후 알림 동의 화면 노출
6. 받은 견적 화면 진입
7. 견적 취소/종료 상태 확인

### 이번 글에서 제외한 범위

- 메인 홈 하단의 추천 고수/광고 카드 영역
- 요청폼 중간 step 전체 스크린샷
- 내부 spec이 없으면 확정할 수 없는 canonical 해석

즉, 이 글은 **관찰 사실 기반의 흐름 정리 + QA 포인트 도출**에 집중합니다.

---

## 3. 플로우를 따라가며 무엇을 관찰했는가

### 3-1. 메인 홈에서 취업 준비 프리셋으로 진입

<img src="./assets/01_home.png" alt="숨고 메인 홈" width="900" />

숨고 메인 홈에서 시작했습니다.  
이번 프로젝트의 관심사는 홈 전체가 아니라, **취업 준비 프리셋을 통해 견적 요청 플로우로 진입하는 흐름**이었습니다.

이 구간에서는 홈 진입, 프리셋 섹션 노출, 실험 콘텐츠 노출, 개인화 타입 할당이 함께 관찰됐습니다.

<details>
<summary>화면에서 관찰된 이벤트 / 핵심 파라미터 보기</summary>

| Event | Observed params |
| --- | --- |
| `View Main` | `Member Type=guest`, `location=others` |
| `view_customer_preset_main_page` | `preset_id[]`, `preset_name[]`, `user_type=guest` |
| `abtest_otg_818_content_display` | `current_page_name=고객홈_페이지`, `user_type=guest` |
| `assign_personalization_type` | `location_page=main`, `purpose=content_personalization`, `type=inactive_user` |

</details>

프리셋 섹션 노출(`view_customer_preset_main_page`)은 `preset_id`, `preset_name`을 배열로 한 번에 수집한다는 점에서, 개별 카드마다 이벤트를 생성하기보다 **프리셋 목록 전체를 한 이벤트로 관리하는 방식**으로 판단됩니다. 이런 구조는 카드별 이벤트를 반복적으로 쌓지 않아도 되고, 프리셋 순서와 같은 변경사항이 생겨도 이벤트명을 다시 나누지 않아도 된다는 점에서 효율적인 로그 설계라고 생각합니다. 만약 index별 또는 카드별 개별 이벤트로 쪼갰다면 노출 순서 변경이나 구성 변경 시 schema 관리가 더 복잡해졌을 가능성을 함께 떠올릴 수 있었습니다.

또한 `abtest_otg_818_content_display`와 개인화 타입 할당(`assign_personalization_type`)이 함께 관찰된다는 점에서, 제가 본 홈 화면 역시 실험 조건이나 개인화 로직이 반영된 하나의 버전이었을 가능성이 있습니다. 이번 raw sample에는 variant를 직접 식별할 수 있는 파라미터가 없어 확정할 수는 없지만, 다른 사용자나 다른 시점에서는 일부 다른 홈 구성이 노출되었을 가능성을 함께 열어두고 해석했습니다.

다만 홈 하단에 있는 추천 카드/광고 영역은 이번 포트폴리오 핵심 흐름과 직접 연결되지 않는다고 판단해, 본문 서술에서는 제외했습니다.

> 즉, 홈 화면을 전부 분석한 것이 아니라 **견적 요청 플로우 진입과 직접 연결되는 이벤트만 선택적으로 추적**했습니다.

---

### 3-2. 취업 준비 프리셋 안에서 스피치 컨설팅 선택

<img src="./assets/02_speech_consulting_entry.png" alt="취업 준비 프리셋 내 스피치 컨설팅 선택 화면" width="900" />

취업 준비 프리셋에 진입한 뒤, 사용자는 세부 서비스 중 **스피치 컨설팅**을 선택해 요청폼으로 들어가게 됩니다.

<details>
<summary>화면에서 관찰된 이벤트 / 핵심 파라미터 보기</summary>

| Event | Observed params |
| --- | --- |
| `customer_total_request_open` | `preset_id=취업 준비`, `preset_name=취업 준비`, `service_count=8`, `total_request_type=preset`, `location=customer_main_page_click_preset_carousel_item` |
| `Start Request Form` | `Form ID`, `Service ID=404`, `Service Name=스피치 컨설팅`, `requestServiceId[]`, `content_category[]`, `content_ids=404`, `form_type=chat_request` |
| `click_request_form_step_next_button` | `request_form_id`, `step_index`, `step_type`, `selected_answer`, `is_last_step` |

</details>

이 흐름에서 `customer_total_request_open`은 **취업 준비 프리셋 페이지에 진입했다는 사실**을 보여주는 이벤트이고, 그 페이지에서 무료견적 받기를 누르면 `Start Request Form`이 켜지는 구조를 확인하였습니다.

또한 `Start Request Form`에는 `requestServiceId`, `content_category`, `content_ids`가 함께 붙어 있어, 단순히 “폼이 열렸다”기보다 **어떤 서비스 맥락으로 폼이 시작됐는지**를 같이 전달하는 구조로 보였습니다. 이어서 `click_request_form_step_next_button`에 `request_form_id`가 붙는다는 점에서, 이 값은 현재 작성 중인 요청폼을 구분하는 식별자로 추측됩니다. 예를 들어 같은 스피치 컨설팅 요청이라도 폼을 연 시점과 작성 흐름은 매번 달라질 수 있기 때문에, `request_form_id`는 “지금 작성 중인 이 폼 묶음”을 식별하는 값처럼 이해하는 것이 자연스러웠습니다. 이때 `step_index`는 질문 의미 자체라기보다, 현재 폼 안에서 사용자가 몇 번째 단계를 진행 중인지 보여주는 순서 값으로 봤습니다.

---

### 3-3. 게스트 사용자의 로그인 게이트 노출

<img src="./assets/03_login_gate.png" alt="숨고 로그인 게이트" width="900" />

이번 관찰은 **게스트 상태에서 시작한 사용자 흐름**이었기 때문에, 제출 직전에 로그인 게이트가 노출됐습니다.

<details>
<summary>화면에서 관찰된 이벤트 / 핵심 파라미터 보기</summary>

| Event | Observed params |
| --- | --- |
| `view_request_sign_in_step` | `sep_device_id`, `soomgo_session_id` |
| `$identify` | `Member Type=user`, `Allow Push=true`, `Allow SMS=true`, `Allow Email=true` |
| `complete_user_sign_in` | `Account Type=Kakao`, `Member Type=user`, `Login Method=manual` |

</details>

이 구간에서는 로그인 게이트 노출(`view_request_sign_in_step`)과 로그인 완료(`complete_user_sign_in`)가 차례로 관찰됐습니다. 이를 통해 숨고의 요청 퍼널은 비로그인 사용자에게 완전히 닫혀 있다기보다, **요청 제출 직전에 인증 단계를 삽입하는 구조**로 읽을 수 있었습니다.

또한 로그인 완료 직전에는 `$identify` 이벤트가 연이어 관찰되며 `Member Type=user`, `Allow Push=true` 같은 사용자 속성이 갱신됐습니다. 즉 이 구간은 단순히 로그인 버튼을 눌렀다는 사실만 남기는 것이 아니라, **게스트 상태의 사용자가 인증을 거치며 회원 상태로 전환되는 과정**까지 함께 기록하는 구간으로 볼 수 있었습니다.

---

### 3-4. 제출 후 알람 동의 화면

<img src="./assets/04_consent.png" alt="알람 동의 화면" width="900" />

요청 제출 이후에는 받은 견적으로 바로 넘어가기 전, 알람 동의 화면이 한 번 더 나타났습니다.

<details>
<summary>관찰 이벤트 / 핵심 파라미터</summary>

| Event | Observed params |
| --- | --- |
| `customer_info_input_start` | `location=request`, `sep_device_id`, `soomgo_session_id` |

</details>

이 구간에서 raw 기준으로 직접 확인된 이벤트는 `customer_info_input_start`였습니다. 실제 화면 흐름을 함께 보면, 이 이벤트는 알람 동의 화면의 시작 이벤트로 해석하는 것이 가장 자연스러웠습니다.

다만 이 화면에서 사용자가 `동의하고 맞춤 콘텐츠 알림 받기`를 눌렀는지, `나중에 받기`를 눌렀는지까지 구분해주는 후속 이벤트는 이번 raw sample에서 명확히 확인되지 않았습니다. 따라서 이 구간은 **알람 동의 화면의 시작은 관찰되지만, 버튼별 후속 액션 계측은 확인되지 않은 구간**으로 정리했습니다.

---

### 3-5. 받은 견적 화면 진입

<img src="./assets/05_received_quotes.png" alt="도착 견적 화면" width="900" />

제출 이후 전환 구간을 지나면, 사용자는 받은 견적 영역으로 이어지게 됩니다.

<details>
<summary>관찰 이벤트 / 핵심 파라미터</summary>

| Event | Observed params |
| --- | --- |
| `view_received_quote_list_page` | `service_id`, `service_name`, `request_id`, `category_name`, `location=requested` |
| `customer_landing_im` | `service_id`, `service_name`, `request_id` |

</details>

이 구간에서는 `view_received_quote_list_page`와 `customer_landing_im`이 연달아 관찰됐고, 두 이벤트 모두 같은 `request_id`, `service_id`, `service_name`을 공유했습니다. 이번 샘플만 놓고 보면 `customer_landing_im`의 파라미터는 `view_received_quote_list_page`의 핵심 식별 정보와 크게 다르지 않아, 이 구간은 **이벤트 통합 또는 역할 재정의가 필요한 후보 포인트**로 정리할 수 있었습니다.

---

## 4. 관찰을 통해 도출한 개선 포인트

#### 4-1. 제출 이벤트 쌍의 존재 이유는 더 명확해질 필요가 있다 _(관련 구간: 3-4)_

`send_request_finished`와 `Submit Request`는 서비스 단위로 짝을 이루며 연속해서 관찰됐습니다. **왜 한 서비스 요청 안에 두 이벤트가 함께 필요한지**는 이번 관찰만으로 충분히 설명되지 않았습니다. 따라서 이 구간은 제출 이벤트의 역할을 더 선명하게 나누거나, 필요하다면 통합 여부까지 검토할 수 있는 포인트로 남았습니다.

#### 4-2. '받은 견적' 진입 이벤트는 통합 검토 후보 _(관련 구간: 3-5)_

`view_received_quote_list_page`와 `customer_landing_im`은 같은 `request_id`, `service_id`, `service_name`을 공유했고, 파라미터 구조도 매우 유사했습니다. 따라서 이 구간은 **이벤트 통합 또는 역할 재정의가 필요한 후보 포인트**로 제시할 수 있었습니다.

#### 4-3. 알람 동의 화면의 후속 액션은 추가 확인이 필요하다 _(관련 구간: 3-4)_

`customer_info_input_start`를 통해 알람 동의 화면의 시작은 확인할 수 있었지만, 사용자가 `동의하고 맞춤 콘텐츠 알림 받기`를 눌렀는지 `나중에 받기`를 눌렀는지까지 설명하는 후속 이벤트는 이번 관찰 데이터에서는 확인되지 않았습니다. 따라서 이 구간은 **버튼별 후속 액션이 실제로 어떻게 계측되는지 추가 확인이 필요한 포인트**로 남았습니다.

#### 4-4. 파라미터 명명 규칙은 표준화 여지 존재 _(관련 구간: 3-1, 3-2, 3-5)_

관찰 과정에서는 `Member Type`과 `user_type`, `Service ID`와 `service_id`/`serviceId`처럼 같은 의미로 보이지만 표기 규칙이 다른 필드들이 함께 나타났습니다. 이런 차이는 raw를 읽는 단계에서는 큰 문제가 아니더라도, 이후 tracking plan 정리나 SQL 기반 검증 단계에서는 혼선을 만들 수 있습니다. 따라서 이 부분은 **파라미터 명명 표준화가 필요한 포인트**로 남았습니다.

---

## 5. 실제 QA에서 검증할 로그 적재 포인트


### 5-1. 서비스 선택 이후 요청폼 시작이 정상적으로 쌓이는지

관련 구간: 3-2

핵심은 프리셋/서비스 진입 이후 `Start Request Form`이 같은 흐름 안에서 정상적으로 이어지는지 확인하는 것입니다.

- 검증 방법: 동일 `soomgo_session_id` 기준으로 `customer_total_request_open` 이후 `Start Request Form`이 이어지는지 확인하고, `requestServiceId`, `content_ids`, `Service ID`가 같은 서비스 맥락을 가리키는지 비교합니다.

### 5-2. 요청 제출 이벤트가 선택한 서비스 기준으로 정확히 생성되는지

관련 구간: 3-4

핵심은 사용자가 선택한 서비스 수만큼 제출 이벤트가 정확히 생성되는지, 그리고 서비스별 request가 올바르게 분리되는지 확인하는 것입니다.

- 검증 방법: 단일/복수 서비스 케이스를 나눠 `send_request_finished`와 `Submit Request`가 서비스 수만큼 생성되는지 확인하고, 각 이벤트의 서비스 식별 필드와 `request_id`가 올바르게 대응하는지 비교합니다.

### 5-3. 제출 이후 받은 견적 진입이 request 기준으로 정상 연결되는지

관련 구간: 3-5

핵심은 요청 제출 이후 사용자가 실제로 같은 request 맥락 안에서 받은 견적 화면으로 이어지는지 확인하는 것입니다.

- 검증 방법: `Submit Request.request_id`를 기준으로 `view_received_quote_list_page`, `customer_landing_im`이 같은 request로 이어지는지 확인합니다.

### 5-4. 비로그인 사용자가 로그인 후에도 제출 퍼널이 끊기지 않는지

관련 구간: 3-3 ~ 3-4

핵심은 게스트 상태에서 시작한 사용자가 로그인 단계를 거친 뒤에도 요청 제출까지 정상적으로 이어지는지 확인하는 것입니다.

- 검증 방법: 동일 `soomgo_session_id`, `sep_device_id` 기준으로 `view_request_sign_in_step` 이후 `complete_user_sign_in`, 제출 이벤트가 같은 세션 안에서 이어지는지 확인합니다.
