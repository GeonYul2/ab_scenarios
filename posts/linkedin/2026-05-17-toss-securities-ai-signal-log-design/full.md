# 2026-05-17 | 토스증권 AI 시그널 탐색 플로우 이벤트 로그 설계 시나리오

> 내부 로그 스키마나 실제 데이터에 접근하지 않고, 앱 화면에서 관찰 가능한 사용자 흐름을 기준으로 **어떤 이벤트와 프로퍼티를 설계하면 분석 맥락을 잃지 않을지** 정리한 시나리오입니다.

---

## 1. 왜 이 프로젝트를 했는가

토스증권 Data Quality Specialist 포지션은 클라이언트 로그 품질 관리, 분석 계획에 맞는 로그 설계, 엔지니어에게 전달 가능한 스펙 작성을 중요하게 봅니다.  
그래서 이번 글에서는 실제 내부 데이터 조회나 입수 확인이 아니라, 공개적으로 관찰 가능한 앱 화면을 바탕으로 **AI 시그널 기능이 종목 탐색으로 이어지는 흐름을 어떻게 이벤트로 남길지**에 집중했습니다.

이번 산출물은 아래 수준까지만 다룹니다.

- 화면별 `view_*` 이벤트
- 다음 화면으로 이어지는 `click_*` 이벤트
- 모든 이벤트에 공통으로 붙일 추적용 envelope
- 각 화면/행동을 해석하는 최소 프로퍼티
- iOS/Android 구현 시 트리거 시점이 달라질 수 있는 유의점

실제 매수, 주문 완료, 내부 로그 입수 여부는 이번 범위에서 제외했습니다.

---

## 2. 관찰 대상 플로우

관찰한 흐름은 **관심 탭에서 AI 시그널을 보고, 국내주식 시그널 상세와 연관 기업을 거쳐 최종 종목 상세로 이동하는 여정**입니다.

| Step | Screen | View event | Transition event |
| --- | --- | --- | --- |
| 1 | 토스증권 메인 | `view_main` | `click_watchlist_tab` |
| 2 | 관심 | `view_watchlist` | `click_ai_signal_entry` |
| 3 | AI 시그널 기본 탭 | `view_ai_signal_list` | `click_ai_signal_tab` |
| 4 | AI 시그널 국내주식 탭 | `view_ai_signal_list` | `click_ai_signal_card` |
| 5 | AI 시그널 상세 | `view_ai_signal_detail` | - |
| 6 | 연관 기업 | `view_related_companies` | `click_related_company` |
| 7 | 검색 결과 | `view_search_result` | `click_stock_search_result` |
| 8 | 종목 상세 | `view_stock_detail` | - |

이번 플로우의 최종 도달 이벤트는 `view_stock_detail`로 두고, 실제 매수 버튼 클릭은 분석 범위에서 제외했습니다.

---

## 3. 공통 이벤트 envelope

개별 이벤트마다 `user_id`, `session_id`를 반복해서 정의하면 이벤트별 프로퍼티가 너무 커집니다.  
따라서 모든 이벤트가 공통으로 갖는 추적용 값은 별도 envelope로 분리하고, 각 이벤트에는 화면/행동 해석에 필요한 값만 남깁니다.

| Property | Meaning |
| --- | --- |
| `event_id` | 이벤트 1건을 식별하는 고유 키 |
| `event_name` | `view_*`, `click_*` 형태의 이벤트명 |
| `event_timestamp` | 이벤트 발생 시각 |
| `user_id` | 비식별 사용자 키 |
| `session_id` | 같은 앱 방문 흐름을 묶는 키 |
| `platform` | `ios`, `android` 등 클라이언트 구분 |
| `app_version` | 앱 버전별 수집 차이를 보기 위한 값 |
| `funnel_id` | `ai_signal_to_stock_detail` |
| `funnel_instance_id` | 한 사용자의 AI 시그널 탐색 1회 여정을 묶는 키 |
| `funnel_step` | 현재 퍼널 단계 |
| `screen_name` | 이벤트가 발생한 논리적 화면명 |
| `previous_event_name` | 직전 이벤트명 |
| `source_event_id` | 현재 이벤트를 유발한 직전 이벤트의 고유 키 |

특히 `source_event_id`와 `funnel_instance_id`는 후반부에서 중요합니다.  
검색 화면처럼 보이는 Image #7이 실제로는 `연관 기업` 클릭에서 온 흐름이기 때문에, 이 연결값이 없으면 최종 `view_stock_detail`의 진입 맥락을 잃을 수 있습니다.

---

## 4. 화면별 이벤트 설계 시나리오

### 4-1. 메인에서 관심 탭으로 이동

<img src="./assets/01_main.png" alt="토스증권 메인 화면" width="360" />

이 화면은 퍼널의 시작점입니다.  
토스 앱에서 증권 서비스에 들어온 사용자가 `내 투자` 영역을 본 뒤, 하단 `관심` 탭으로 이동하는 흐름을 잡습니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_main` | `selected_tab=증권`, `is_my_investment_visible=true`, `is_holding_stock_visible=true` |
| `click_watchlist_tab` | `source_module=bottom_navigation`, `from_tab_name=증권`, `to_tab_name=관심`, `target_event_name=view_watchlist` |

</details>

이 단계에서는 계좌 잔액, 광고, 주문내역처럼 화면에 보이는 모든 정보를 담기보다, 이번 퍼널의 출발 조건인 `내 투자` 노출과 관심 탭 이동 여부만 남기는 것이 적절하다고 봤습니다.

---

### 4-2. 관심 화면에서 AI 시그널 진입

<img src="./assets/02_watchlist.png" alt="토스증권 관심 화면" width="360" />

관심 화면에는 관심 종목, 추천 주식, AI 시그널 진입점이 함께 보입니다.  
이 단계의 핵심은 사용자가 단순히 관심 화면만 보는지, 아니면 `토스증권 AI 시그널` 영역으로 실제 진입하는지 구분하는 것입니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_watchlist` | `selected_tab=관심`, `is_ai_signal_entry_visible=true`, `visible_content_groups=[ai_signal_entry, watchlist_stocks, recommended_stocks]` |
| `click_ai_signal_entry` | `source_module=ai_signal_entry`, `entry_label=토스증권 AI 시그널`, `target_event_name=view_ai_signal_list`, `target_tab_name=내 투자·관심` |

</details>

`visible_content_groups`를 남기면 AI 시그널 진입점이 보였는데도 관심 종목만 보고 끝난 사용자를 구분할 수 있습니다.

---

### 4-3. AI 시그널 기본 탭에서 시장 탭으로 이동

<img src="./assets/03_ai_signal_default.png" alt="AI 시그널 내 투자 관심 탭" width="360" />

AI 시그널에 진입하면 기본으로 `내 투자·관심` 탭이 보입니다.  
여기서는 사용자가 개인화된 시그널에 머무는지, `국내주식`이나 `해외주식`으로 탐색 범위를 넓히는지 보는 것이 핵심입니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_ai_signal_list` | `tab_name=내 투자·관심`, `signal_context_type=personalized`, `available_tab_names=[내 투자·관심, 국내주식, 해외주식]`, `visible_signal_count=5` |
| `click_ai_signal_tab` | `from_tab_name=내 투자·관심`, `to_tab_name=국내주식`, `target_event_name=view_ai_signal_list` |

</details>

탭 이동은 별도 `select_*` 이벤트를 만들기보다 `click_ai_signal_tab`에 `from_tab_name`, `to_tab_name`을 담는 방식이 더 단순합니다.

---

### 4-4. 국내주식 탭에서 시그널 카드 클릭

<img src="./assets/04_ai_signal_domestic.png" alt="AI 시그널 국내주식 탭" width="360" />

`국내주식` 탭에서는 여러 시그널 카드가 리스트로 보입니다.  
이 단계에서는 시장 탭 이동이 실제 시그널 상세 소비로 이어지는지를 봅니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_ai_signal_list` | `tab_name=국내주식`, `market_scope=domestic`, `visible_signal_count=6` |
| `click_ai_signal_card` | `tab_name=국내주식`, `stock_name=현대해상`, `card_position=1`, `signal_summary=1분기 실적 개선으로`, `signal_direction=up`, `signal_change_rate=2.1`, `target_event_name=view_ai_signal_detail` |

</details>

`card_position`을 남기면 첫 번째 카드와 하단 카드의 반응 차이를 볼 수 있고, `signal_summary`는 어떤 메시지가 상세 진입을 유도했는지 해석하는 데 도움을 줍니다.

---

### 4-5. AI 시그널 상세에서 설명 영역 노출

<img src="./assets/05_ai_signal_detail.png" alt="AI 시그널 상세 설명" width="360" />

시그널 카드를 누르면 바텀시트 형태의 상세 화면이 열리고, `왜 올랐을까?` 설명이 보입니다.  
이 화면은 사용자가 AI가 요약한 이유 설명을 실제로 볼 수 있는 상태가 되었는지를 표현합니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_ai_signal_detail` | `stock_name=현대해상`, `source_tab_name=국내주식`, `signal_summary=1분기 실적 개선으로`, `signal_direction=up`, `signal_change_rate=2.1`, `is_reason_section_visible=true`, `is_related_companies_visible=false` |

</details>

스크롤 자체를 이벤트로 두면 해석이 복잡해질 수 있으므로, 이번 시나리오에서는 연관 기업 영역이 실제로 화면에 들어온 순간을 다음 `view_related_companies`로 분리했습니다.

---

### 4-6. 연관 기업 영역에서 다른 기업 클릭

<img src="./assets/06_related_companies.png" alt="AI 시그널 상세 연관 기업" width="360" />

설명 영역 아래에는 `연관 기업` 섹션이 보입니다.  
이 단계는 사용자가 원래 종목 설명에서 끝나는지, 연관 기업을 통해 다른 종목 탐색으로 확장하는지 보는 구간입니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_related_companies` | `source_stock_name=현대해상`, `section_name=연관 기업`, `visible_related_company_count=2`, `first_related_stock_name=코리안리` |
| `click_related_company` | `source_module=related_companies`, `source_stock_name=현대해상`, `section_name=연관 기업`, `related_stock_name=코리안리`, `related_stock_position=1`, `attribution_context=ai_signal_related_company`, `target_event_name=view_search_result` |

</details>

여기서 `attribution_context`가 중요합니다.  
나중에 검색 결과나 종목 상세 화면만 보면 일반 검색처럼 보일 수 있지만, 실제로는 AI 시그널 상세의 연관 기업에서 시작된 탐색이기 때문입니다.

---

### 4-7. 검색 결과 화면으로 연결

<img src="./assets/07_search_result.png" alt="코리안리 검색 결과" width="360" />

연관 기업을 누르면 검색 결과처럼 보이는 화면으로 이동합니다.  
이 화면은 검색 UX로 보이지만, 분석 관점에서는 `AI 시그널 → 연관 기업 → 검색 결과`라는 맥락이 유지되어야 합니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_search_result` | `query=코리안리`, `entry_point=ai_signal_related_company`, `source_stock_name=현대해상`, `result_stock_name=코리안리`, `is_stock_result_visible=true` |
| `click_stock_search_result` | `query=코리안리`, `entry_point=ai_signal_related_company`, `stock_name=코리안리`, `stock_market=domestic`, `result_position=1`, `target_event_name=view_stock_detail` |

</details>

`entry_point`가 없다면 이 사용자가 직접 검색창에서 코리안리를 검색한 것인지, AI 시그널 연관 기업을 통해 이동한 것인지 구분하기 어렵습니다.

---

### 4-8. 최종 종목 상세 도달

<img src="./assets/08_stock_detail.png" alt="코리안리 종목 상세" width="360" />

최종적으로 사용자는 코리안리 종목 상세 화면에 도달합니다.  
이번 퍼널에서는 이 화면 진입을 최종 도달 이벤트로 보고, 구매 버튼 클릭 이후의 거래 행동은 제외합니다.

<details>
<summary>이벤트 / 핵심 프로퍼티</summary>

| Event | Event-specific properties |
| --- | --- |
| `view_stock_detail` | `stock_name=코리안리`, `stock_market=domestic`, `landing_tab_name=차트`, `entry_point=search_result`, `original_entry_point=ai_signal_related_company`, `source_stock_name=현대해상`, `is_funnel_conversion=true` |

</details>

`entry_point`는 직전 화면 기준으로 `search_result`이고, `original_entry_point`는 전체 퍼널 기준으로 `ai_signal_related_company`입니다.  
이 둘을 분리해야 종목 상세 도달의 직접 경로와 원래 탐색 맥락을 함께 설명할 수 있습니다.

---

## 5. 이 설계로 보고 싶은 질문

이번 이벤트 설계는 아래 질문에 답하기 위한 최소 구조입니다.

1. 관심 화면 사용자가 AI 시그널 진입점까지 실제로 이동하는가?
2. AI 시그널 진입 후 `내 투자·관심`에 머무는가, 시장 탭으로 탐색을 넓히는가?
3. `국내주식` 탭의 시그널 카드는 상세 화면 소비로 이어지는가?
4. 사용자는 AI 요약 설명만 보고 끝나는가, `연관 기업` 영역까지 내려가는가?
5. 연관 기업 클릭은 검색 결과와 최종 종목 상세 도달로 이어지는가?
6. 최종 종목 상세 도달은 일반 검색이 아니라 AI 시그널 기반 탐색으로 귀속될 수 있는가?

---

## 6. 구현 시 중점적으로 맞춰야 할 부분

실제 로그를 설계한다면, 먼저 아래 정의가 맞아야 한다고 생각했습니다.

- iOS와 Android에서 `view_*`가 화면 로드 완료 시점에 찍히는지, 클릭 직후 선행해서 찍히는지 맞춰야 합니다.
- `click_*`가 실제 탭 순간에 찍히는지, 다음 화면 진입 성공 후 찍히는지에 따라 이탈 해석이 달라집니다.
- 바텀시트인 `view_ai_signal_detail`, `view_related_companies`는 노출 기준을 “시트 오픈”으로 볼지, “해당 섹션이 화면 안에 들어온 시점”으로 볼지 정해야 합니다.
- 앱 재진입, 뒤로가기, 탭 재클릭, 화면 재렌더링에서 `view_*`가 중복으로 쌓일 수 있으므로 `event_id`, `source_event_id`, `funnel_instance_id` 연결 기준이 중요합니다.
- 검색 결과 화면처럼 보이지만 실제 진입 경로가 연관 기업인 경우, `entry_point`와 `original_entry_point`가 없으면 최종 종목 상세의 귀속이 흐려집니다.

---

## 7. 정리

이번 글은 토스증권 내부 로그를 추정하거나 실제 데이터로 성과를 검증한 글이 아닙니다.  
대신 관찰 가능한 앱 화면을 기준으로, 분석자가 보고 싶은 질문이 이벤트명과 프로퍼티에 어떻게 반영되어야 하는지 정리했습니다.

특히 이번 플로우에서 가장 중요했던 기준은 두 가지였습니다.

1. 공통 추적값은 envelope로 분리하고, 이벤트별 프로퍼티는 최소화한다.
2. 최종 종목 상세 도달이 어떤 탐색 맥락에서 왔는지 잃지 않도록 `source_event_id`, `entry_point`, `original_entry_point`를 남긴다.

이 정도만 명확해도 “사용자가 AI 시그널을 봤다”에서 끝나지 않고, **AI 시그널이 실제 종목 탐색 행동으로 이어졌는지**를 읽을 수 있는 로그 설계 시나리오가 됩니다.
