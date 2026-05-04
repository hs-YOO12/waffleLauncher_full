# 파이썬 실무: 견고한(Robust) 커맨드 파싱 가이드

이 자료는 로봇 제어나 하드웨어 통신 프로토콜을 처리할 때, 단순한 파싱을 넘어 발생 가능한 모든 예외 상황을 체계적으로 관리하는 방법을 다룹니다.

---

## 1. 커맨드 파싱 에러 컨벤션 (Error Convention)

에러 발생 시 단순한 `False` 반환이 아닌, 구체적인 에러 코드와 메시지를 제공하여 디버깅 효율을 극대화합니다.

| 에러 코드 | 단계 | 메시지 예시 | 발생 조건 |
| :--- | :--- | :--- | :--- |
| **[LEN_ERR]** | 구조 검증 | Expected 5 tokens, but got {n}. | 구분자로 분리된 토큰 수가 5개가 아님 |
| **[MARK_ERR]** | 마커 검증 | Invalid Start/End marker. ('H', 'E' expected) | 첫 토큰이 'H'가 아니거나 마지막이 'E'가 아님 |
| **[TYPE_ERR]** | 형식 검증 | Invalid motor format '{val}'. (F1~3, B1~3 expected) | 모터 문자열이 'F1' 등의 형식이 아님 |
| **[TYPE_ERR]** | 형식 검증 | Angle '{val}' is not a valid integer. | 각도 자리에 숫자가 아닌 문자가 들어옴 |
| **[RANGE_ERR]** | 범위 검증 | Motor speed {n} out of range (0~3). | 모터 숫자가 0보다 작거나 3보다 큼 |
| **[RANGE_ERR]** | 범위 검증 | Angle {n} out of range (50~250). | 각도가 50 미만이거나 250 초과임 |

---

## 2. 정제된 파이썬 코드 예시

이 코드는 구체적인 에러 메시지를 반환하여 상위 모듈에서 어떤 문제가 발생했는지 즉각적으로 파악할 수 있게 설계되었습니다.

```python
import re

def normalize_tokens(tokens):
    """공백 제거 및 대문자 변환"""
    return [t.strip().upper() for t in tokens if t.strip()]

def split_command(data):
    """구분자(콤마 또는 공백)로 분리"""
    return re.split(r'[,\s]+', data)

def parse_h_command_refined(data: str):
    tokens = normalize_tokens(split_command(data))
    
    # 1. 길이 검증
    if len(tokens) != 5:
        return f"[LEN_ERR] Expected 5 tokens, but got {len(tokens)}: {tokens}"

    # 2. 마커(H, E) 검증
    if tokens[0] != 'H' or tokens[4] != 'E':
        return f"[MARK_ERR] Invalid format. Must start with 'H' and end with 'E'. (Current: {tokens[0]}, {tokens[4]})"

    # 3. 모터 데이터(Body) 검증
    motor_pattern = re.compile(r'^([FB])(\d)$')
    for i, label in enumerate(['right_motor', 'left_motor'], start=1):
        match = motor_pattern.match(tokens[i])
        if not match:
            return f"[TYPE_ERR] Invalid {label} format: '{tokens[i]}'. Use F0~F3 or B0~B3."
        
        direction, speed = match.groups()
        if not (0 <= int(speed) <= 3):
            return f"[RANGE_ERR] {label} speed '{speed}' out of range (0~3)."

    # 4. 각도 데이터(Body) 검증
    angle_str = tokens[3]
    if not angle_str.lstrip('-').isdigit():
        return f"[TYPE_ERR] Angle '{angle_str}' is not an integer."
    
    angle = int(angle_str)
    if not (50 <= angle <= 250):
        return f"[RANGE_ERR] Angle '{angle}' out of range (50~250)."

    # 성공 시 결과 반환
    return {
        'type': 'H',
        'right_motor': tokens[1],
        'left_motor': tokens[2],
        'angle': angle
    }
```

---

## 3. 상황별 에러 출력 결과 (Test Cases)

코드 적용 시 다음과 같은 명확한 피드백을 얻을 수 있습니다.

- **잘못된 모터 범위** (H, F4, F3, 150, E)
  - `[RANGE_ERR] right_motor speed '4' out of range (0~3).`
- **잘못된 각도 범위** (H, F1, B2, 45, E)
  - `[RANGE_ERR] Angle '45' out of range (50~250).`
- **잘못된 데이터 타입** (H, F1, B2, ABC, E)
  - `[TYPE_ERR] Angle 'ABC' is not an integer.`
- **구조적 오류** (H, F1, B2, 150)
  - `[LEN_ERR] Expected 5 tokens, but got 4: ['H', 'F1', 'B2', '150']`

---

## 4. 실습 과제: LED 제어기(L-Command) 구현

위의 컨벤션을 적용하여 다음 명령어를 파싱하는 함수를 작성해 보세요.

- **형식**: `L [R] [G] [B] E` (예: `L 255 128 0 E`)
- **제한 사항**: R, G, B는 각각 0~255 사이의 정수여야 함.
