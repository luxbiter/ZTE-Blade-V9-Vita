# ZTE Blade V9 Vita (P840F03) LED 제어 가이드

> 기기: SKT ZTE Blade V9 Vita / P840F03  
> 환경: Android 8 계열, 루트 셸 (`su`)  
> 작성 기준: **실제 기기에서 확인한 실험 결과**. 드라이버 내부 구조는 별도로 확인하지 않았음.

## 1. 개요

이 기기는 `/sys/class/leds/` 아래에 **빨간색(`red`)과 초록색(`green`) LED**가 노출돼 있다. 두 색을 동시에 켜면 육안상 **주황색**이 된다.

- 두 색의 ON/OFF는 개별적으로 제어 가능하다.
- `brightness`는 0~255를 받아들이고 읽기 값도 유지하지만, 실험에서 **수치에 따른 밝기 차이는 육안상 확인되지 않았다**. 단순 ON/OFF 출력일 가능성이 있다.
- `blink`, `blink_on`, `blink_off`로 자동 점멸을 켜고 끄거나 주기를 바꿀 수 있으며, **실제로 동작함을 확인했다**.
- 두 색을 **서로 다른 자동 점멸 속도로 동시에** 설정하면 예상과 달리 같은 일반 속도로 점멸하는 **주황색 LED**가 나타났다. 반면 한쪽을 상시 ON, 다른 쪽을 점멸로 설정하는 것은 가능하다.
- 셸 스크립트로 밝기를 번갈아 쓰면 **빨강/초록 독립 패턴**을 구현할 수 있다.

## 2. LED 경로와 현재까지 확인된 파일

```text
/sys/class/leds/
├── red/
│   ├── brightness
│   ├── max_brightness
│   ├── trigger
│   ├── blink
│   ├── blink_on
│   └── blink_off
└── green/
    ├── brightness
    ├── max_brightness
    ├── trigger
    ├── blink
    ├── blink_on
    └── blink_off
```

두 LED의 `device` 링크가 `../../../soc:gpio_leds`를 가리킨다. `max_brightness`는 둘 다 `255`였다.

초기 확인값:

| 항목 | red | green |
|---|---|---|
| `brightness` | `255` | `255` |
| `trigger` | `[none]` | `[none]` |
| `blink` | `0` | 미확인 |
| `blink_on` | `500` | 미확인 |
| `blink_off` | `1500` | 미확인 |

`trigger`의 대괄호는 **현재 선택된 트리거**를 뜻한다. 확인된 선택지:

```text
[none] usb-online mmc0 mmc1 ti_bms-online battery-charging-or-full
battery-charging battery-full battery-charging-blink-full-solid
bcl-online rfkill0
```

> `trigger`의 항목들은 커널에서 제공하는 자동 트리거 이름이다. 각 트리거가 이 기기에서 어떤 상황에 정확히 반응하는지는 아직 실험하지 않았다.

## 3. 시작 전 안전 확인

**모든 명령은 기기의 루트 셸에서 실행한다.** ADB를 사용할 경우:

```sh
adb shell
su
```

설정을 바꾸기 전에 값을 기록한다:

```sh
cat /sys/class/leds/red/brightness
cat /sys/class/leds/green/brightness
cat /sys/class/leds/red/trigger
cat /sys/class/leds/green/trigger
cat /sys/class/leds/red/blink
cat /sys/class/leds/green/blink
cat /sys/class/leds/red/blink_on
cat /sys/class/leds/red/blink_off
cat /sys/class/leds/green/blink_on
cat /sys/class/leds/green/blink_off
```

충전 상태나 알림 서비스가 값을 다시 쓸 수 있다. 실험 후에는 필요에 따라 원래 설정을 복원하고, **충전 표시 LED가 정상 작동하는지** 확인한다.

## 4. 색상 제어

### 빨강만 켜기

```sh
echo 0 > /sys/class/leds/green/brightness
echo 255 > /sys/class/leds/red/brightness
```

### 초록만 켜기

```sh
echo 0 > /sys/class/leds/red/brightness
echo 255 > /sys/class/leds/green/brightness
```

### 주황색 (빨강 + 초록)

```sh
echo 255 > /sys/class/leds/red/brightness
echo 255 > /sys/class/leds/green/brightness
```

### 모두 끄기

```sh
echo 0 > /sys/class/leds/red/brightness
echo 0 > /sys/class/leds/green/brightness
```

| 빨강 | 초록 | 관찰된 색 |
|---|---|---|
| OFF | OFF | 꺼짐 |
| ON | OFF | 빨강 |
| OFF | ON | 초록 |
| ON | ON | 주황 |

**밝기 테스트:** `echo 1 > /sys/class/leds/red/brightness` 직후 `cat` 명령은 `1`을 반환했고, `255` 입력 후에는 `255`를 반환했다. 하지만 밝기 변화는 눈에 띄지 않았다. 따라서 **설정값 보존 ≠ 실제 밝기 256단계 지원**이다.

## 5. 자동 점멸 (실제 검증됨)

### 기본으로 확인한 패턴: 500ms ON / 1500ms OFF

```sh
echo 0 > /sys/class/leds/red/blink
echo 500 > /sys/class/leds/red/blink_on
echo 1500 > /sys/class/leds/red/blink_off
echo 1 > /sys/class/leds/red/blink
```

실제 빨간 LED가 **0.5초 켜짐, 1.5초 꺼짐**으로 동작했다.

### 빠른 패턴: 200ms ON / 200ms OFF

```sh
echo 0 > /sys/class/leds/red/blink
echo 200 > /sys/class/leds/red/blink_on
echo 200 > /sys/class/leds/red/blink_off
echo 1 > /sys/class/leds/red/blink
```

실제로 빠른 점멸이 되는 것까지 확인했다.

### 자동 점멸 중지

```sh
echo 0 > /sys/class/leds/red/blink
```

점멸을 멈춰도 LED가 현재 상태로 남을 수 있으니 필요하면 `brightness`도 설정한다.

### 초록 LED에 적용

경로의 `red`를 `green`으로 바꾼다. 예:

```sh
echo 0 > /sys/class/leds/green/blink
echo 500 > /sys/class/leds/green/blink_on
echo 1500 > /sys/class/leds/green/blink_off
echo 1 > /sys/class/leds/green/blink
```

## 6. 두 색을 조합할 때의 특이사항

실험에서:

1. 빨강 200ms/200ms, 초록 500ms/1500ms처럼 **서로 다른 자동 점멸 주기를 동시에 적용**해도 개별 속도로 점멸하지 않았다. **같은 일반 속도로 점멸하는 주황색**이 됐다.
2. **초록 LED의 `blink_on/off`만 변경하고 초록 점멸은 켜지 않은 상태**에서는 빨간 LED의 기존 점멸 속도가 변하지 않았다.
3. 한 색 **상시 ON** + 다른 색 **자동 점멸** 조합은 작동했다. 예를 들어 초록 상시 ON + 빨강 점멸이면 **초록 ↔ 주황**으로 보인다.
4. 셸에서 각 LED의 `brightness`를 직접 변경하는 **독립 색상 연출은 작동했다**.

**해석:** `blink_on/off` 설정값은 적어도 단독 설정 시 서로 간섭하지 않지만, 두 채널의 자동 점멸을 동시에 켰을 때에는 동기화 또는 공통 제어 현상이 나타난다. **드라이버 코드나 회로도를 확인하지 않았으므로 공유 하드웨어 타이머라고 단정할 수 없다.**

## 7. 빨강 ↔ 초록 교대 점멸 (셸에서 검증됨)

커널 자동 점멸을 양쪽 모두 끄고, 셸 루프로 직접 제어한다.

```sh
echo 0 > /sys/class/leds/red/blink
echo 0 > /sys/class/leds/green/blink

while true; do
    echo 255 > /sys/class/leds/red/brightness
    echo 0 > /sys/class/leds/green/brightness
    sleep 0.5

    echo 0 > /sys/class/leds/red/brightness
    echo 255 > /sys/class/leds/green/brightness
    sleep 0.5
done
```

- 중지는 터미널에서 **Ctrl+C**.
- 멈춘 뒤 마지막 색이 남아 있을 수 있으므로 필요하면 두 `brightness`를 `0`으로 설정한다.
- 셸 루프는 커널 자체 점멸보다 CPU를 주기적으로 깨우므로 장시간 사용 시 배터리와 프로세스 관리에 주의한다.
- 터미널을 닫으면 루프가 종료될 수 있다.

## 8. 실험 종료 및 원상 복구

임시로 LED를 끄려면:

```sh
echo 0 > /sys/class/leds/red/blink
echo 0 > /sys/class/leds/green/blink
echo 0 > /sys/class/leds/red/brightness
echo 0 > /sys/class/leds/green/brightness
```

**주의:** 이 명령은 *공장 초기 설정 복원*이 아니라 *양쪽 LED 점멸 중단 및 끄기*다. 충전/알림 동작까지 포함해 원래대로 복원하려면 **3절에서 기록해 둔 트리거와 값**을 참고한다. 안드로이드 서비스가 원래 LED를 다시 설정하기도 한다.

## 9. 추후 연구 과제

- 개별 밝기 값 `1`과 `255`의 실제 광량 차이 정밀 측정 / PWM 지원 여부.
- 두 채널 자동 점멸 동기화 원인(드라이버 구현, GPIO 제어 구조).
- 커널의 `trigger`를 통한 충전/배터리 상태 자동 표시 동작.
- 부팅이나 알림 이벤트에 따라 LED 제어를 자동 실행하는 방법. 현재 Magisk **16.7** 환경이므로 최신 Magisk의 `service.d` 사용법을 그대로 적용하기 전에 지원 여부를 검증해야 한다.

---

**핵심 요약:** **빨강 / 초록 / 주황**, 개별 ON/OFF, 커널 자동 점멸 속도 변경, 셸 독립 패턴은 확인됐다. **서로 다른 속도의 동시 자동 점멸**은 현재 실험 환경에서 독립적으로 작동하지 않았다.
