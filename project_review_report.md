# 코드 리뷰 리포트

## simulator.py (208줄)

### 스타일 검사
다음은 PEP 8 및 일반적인 스타일 규칙 위반 사항입니다:

**네이밍 규칙 위반**
- [17] 네이밍 규칙: 딕셔너리 리스트 `vehicles`는 모듈 수준 변수지만 상수처럼 사용되지 않으므로 괜찮으나, 전역 변수 `cleansed_list`, `anomaly_list`, `list_lock`은 모듈 수준의 변경 가능한 전역 상태로 사용되는데, 이에 대한 명시적 주석이나 캡슐화가 없음

**매직 넘버(Magic Number) 사용**
- [77] 스타일: `if len(vehicle["msg_times"]) >= 5` — `5`는 매직 넘버로, 상수(`MAX_MSG_PER_SECOND = 5`)로 분리해야 함
- [80] 스타일: `speed_diff >= 50` 및 `speed_diff <= -50` — `50`은 매직 넘버로 상수로 분리해야 함
- [85] 스타일: `raw_data["fuel_level"] < 5` — `5`는 매직 넘버
- [89] 스타일: `lat_diff > 1` — `1`은 매직 넘버
- [130] 스타일: `roll < 0.05`, `roll < 0.08`, `roll < 0.10` — 모두 매직 넘버로 상수로 정의해야 함
- [155] 스타일: `random.random() < 0.15`, `random.random() < 0.7` — 매직 넘버
- [159] 스타일: `random.uniform(0.1, 0.5)`, `return 1, 1, 0.5`, `return 2, 1, 1`, `return 3, 2, 1.5` — 모두 매직 넘버

**함수 설계 및 복잡도**
- [71] 스타일: `check_anomaly` 함수가 여러 이상 유형을 한꺼번에 검사하여 단일 책임 원칙(SRP) 위반, 함수 분리 권장
- [106] 스타일: `integrated_processor` 함수가 lock 획득과 로직 처리를 혼합하여 가독성 저하
- [150] 스타일: `update_vehicle_status` 함수가 튜플 `(mode, event_type, interval)`을 반환하는데, 반환 타입에 대한 타입 힌트나 주석 없음

**타입 힌트 누락**
- [34], [38], [52], [62], [71], [97], [106], [120], [150], [167], [176] 스타일: 모든 함수에 타입 힌트(type hint)가 없음 (PEP 3107 / PEP 484 권장)

**docstring 누락**
- [34], [38], [52], [62], [71], [97], [106], [120], [150], [167], [176] 스타일: 모든 함수에 docstring이 없음 (PEP 257 권장)

**파일 핸들링**
- [185] 스타일: `open(CLEANSED_FILE, "w").close()` — 파일 객체를 `with` 문 없이 사용하여 명시적 리소스 관리가 되지 않음. `with open(...) as f: pass` 형태 권장

**인라인 주석 및 가독성**
- [101] 스타일: `print(f"[정제:{len(cleansed_list)}] [이상:{len(anomaly_list)}]", end="\r")` — 디버깅용 print문이 함수 내에 하드코딩됨, 로깅 모듈 사용 권장

**전역 변수 사용**
- [12~14] 스타일: `cleansed_list`, `anomaly_list`, `list_lock`이 전역 변수로 선언되어 있으며 여러 함수에서 직접 접근하고 있음. 클래스로 캡슐화하거나 명시적으로 인자로 전달하는 것이 권장됨

**문자열 리터럴**
- [32] 스타일: `save_line` 함수 내 파일을 append 모드(`"a"`)로 열지만, 파일 초기화와 쓰기 로직이 분리되어 있어 혼동 가능성 있음 — 주석으로 명시 권장

### 보안 검사
코드를 분석한 결과, 요청하신 SQL Injection, XSS, 하드코딩된 비밀번호는 발견되지 않았습니다. 해당 코드는 웹 애플리케이션이나 데이터베이스를 사용하지 않는 로컬 시뮬레이션 코드이기 때문입니다.

다만, 아래와 같은 보안 및 코드 품질 취약점이 존재합니다.

---

## 🔴 심각도: 높음

### 1. 입력값 검증 부재 (Insufficient Input Validation)
- **위치**: `check_anomaly()` 함수, `save_cleansed_data()` 함수
- **유형**: 신뢰할 수 없는 데이터 처리
- **설명**: `raw_data["speed"]`, `raw_data["fuel_level"]`, `raw_data["lat"]` 등의 값에 대해 키 존재 여부 확인 없이 직접 접근합니다. `MISSING_DATA` 케이스에서 `speed` 키가 없는 payload가 들어오면 `KeyError`가 발생하며, 이는 서비스 중단(DoS)으로 이어질 수 있습니다.
- **수정 제안**:
```python
speed = raw_data.get("speed", 0)
fuel_level = raw_data.get("fuel_level", 0)
lat = raw_data.get("lat", 0)
```

---

## 🟠 심각도: 중간

### 2. Race Condition (경쟁 조건)
- **위치**: `integrated_processor()` 함수
- **유형**: 스레드 안전성 문제
- **설명**: `check_anomaly()` 호출은 `list_lock`으로 보호되지만, `vehicle["last_speed"]`와 `vehicle["last_lat"]` 업데이트는 락 밖에서 이루어집니다. 여러 스레드가 동일한 vehicle 딕셔너리를 수정할 경우 데이터 불일치가 발생할 수 있습니다.
- **수정 제안**: vehicle 상태 업데이트도 동일한 락 블록 안으로 포함시키거나, 차량별 개별 락을 사용합니다.
```python
with list_lock:
    if check_anomaly(vehicle, raw_data):
        return
    vehicle["last_speed"] = raw_data["speed"]
    vehicle["last_lat"] = raw_data["lat"]
```

### 3. 파일 핸들 미닫힘 (Unclosed File Handle)
- **위치**: `__main__` 블록
```python
open(CLEANSED_FILE, "w").close()
open(ANOMALY_FILE, "w").close()
```
- **유형**: 리소스 누수
- **설명**: `open()`의 반환값이 즉시 `.close()` 되고 있으나, 예외 발생 시 파일이 닫히지 않을 수 있습니다.
- **수정 제안**:
```python
with open(CLEANSED_FILE, "w") as f:
    pass
with open(ANOMALY_FILE, "w") as f:
    pass
```

---

## 🟡 심각도: 낮음

### 4. 민감 정보 로깅 가능성
- **위치**: `save_line()`, `add_alert()`, `save_cleansed_data()` 함수
- **유형**: 민감 데이터 노출 (OWASP A09 - Security Logging and Monitoring Failures)
- **설명**: `driver_id`, GPS 좌표(`lat`, `lon`), 연료 수준 등 개인 식별 가능 정보(PII)가 평문으로 파일에 저장됩니다. 파일 접근 권한이 적절히 관리되지 않으면 데이터 유출 위험이 있습니다.
- **수정 제안**: 파일 저장 시 민감 필드를 마스킹하거나 암호화하고, 파일 시스템 접근 권한을 최소화합니다.

### 5. TARGET_COUNT 동시성 체크 미흡
- **위치**: `add_alert()`, `save_cleansed_data()` 함수
- **유형**: TOCTOU (Time-of-Check to Time-of-Use)
- **설명**: `len(anomaly_list) >= TARGET_COUNT` 체크 후 실제 append까지 사이에 다른 스레드가 개입하면 TARGET_COUNT를 초과할 수 있습니다.
- **수정 제안**: 해당 체크와 append를 동일한 락 블록 내에서 처리합니다.

---

## ✅ 요청 항목 결론

| 취약점 유형 | 존재 여부 | 비고 |
|---|---|---|
| SQL Injection | ❌ 없음 | DB 미사용 |
| XSS | ❌ 없음 | 웹 환경 아님 |
| 하드코딩된 비밀번호 | ❌ 없음 | 인증 정보 없음 |
| 입력값 검증 부재 | ✅ 있음 | 높음 |
| Race Condition | ✅ 있음 | 중간 |
| 파일 핸들 관리 | ✅ 있음 | 중간 |
| 민감 정보 평문 저장 | ✅ 있음 | 낮음 |

---

