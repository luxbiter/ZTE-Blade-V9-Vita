# ZTE Blade V9 Vita (P840F03) 내장 배경화면 커스터마이징

> 실험 환경: ZTE P840F03 / Android 8 계열 / Magisk 16.7 (구형 Magic Mount)
> 
> 이 문서는 **기기에서 직접 확인된 결과**와 아직 확인하지 않은 내용을 구분해 기록한 간단한 참고용 문서다.

## 확인된 구조

내장 배경화면 경로:

```text
/system/etc/custom_config/app/Launcher/
├── config.xml
├── default_wallpaper.jpg
└── wallpapers/
    ├── 001.jpg
    ├── 001_small.jpg
    ├── ...
    ├── 015.jpg
    ├── 015_small.jpg
    ├── 016.jpg          # 직접 추가 성공
    └── 016_small.jpg    # 직접 추가 성공
```

- `NNN.jpg`: 적용할 배경화면 이미지
- `NNN_small.jpg`: 배경화면 목록의 미리보기 이미지
- `default_wallpaper.jpg`: 기본 배경화면으로 추정되는 별도 파일
- `config.xml`: 런처 설정 파일. 확인된 내용에는 배경화면 파일 목록이 없음

원래 `001`~`015`의 15쌍이 있었으며, **`015` 이미지/썸네일 교체와 `016` 이미지/썸네일 추가가 실제로 성공했다.** `WallpaperPicker_stock.apk`를 다시 빌드하거나 `/system` 파티션을 직접 수정하지 않아도 됐다.

## 1. 원본 이미지 PC에 추출

Windows PowerShell에서 ADB 연결 후:

```powershell
adb pull /system/etc/custom_config/app/Launcher/wallpapers/ ZTE-Wallpapers
adb pull /system/etc/custom_config/app/Launcher/default_wallpaper.jpg
```

원본은 수정하지 않고 백업으로 남겨 둔다.

## 2. 교체 또는 추가 이미지 준비

- 기존 15번 교체: `015.jpg` + `015_small.jpg`
- 새 16번 추가: `016.jpg` + `016_small.jpg`
- 두 파일 모두 실제 **JPEG**로 저장한다. 썸네일은 작은 크기로 만들되 원본 썸네일의 해상도와 비율을 참고한다.

PC에서 휴대폰으로 복사하는 예:

```powershell
adb push 016.jpg /sdcard/Download/016.jpg
adb push 016_small.jpg /sdcard/Download/016_small.jpg
```

## 3. Magisk 16.7 모듈에 배치

이 기기에서는 구형 Magisk 모듈이 `/sbin/.core/img/`에 위치한다. 이미 생성해 사용 중인 `zte_wallpapers` 모듈을 기준으로 한다.

루트 셸에서 **한 줄씩**:

```sh
MOD=/sbin/.core/img/zte_wallpapers
mkdir -p "$MOD/system/etc/custom_config/app/Launcher/wallpapers"
cp /sdcard/Download/016.jpg "$MOD/system/etc/custom_config/app/Launcher/wallpapers/016.jpg"
cp /sdcard/Download/016_small.jpg "$MOD/system/etc/custom_config/app/Launcher/wallpapers/016_small.jpg"
chown 0:0 "$MOD/system/etc/custom_config/app/Launcher/wallpapers/016.jpg" "$MOD/system/etc/custom_config/app/Launcher/wallpapers/016_small.jpg"
chmod 644 "$MOD/system/etc/custom_config/app/Launcher/wallpapers/016.jpg" "$MOD/system/etc/custom_config/app/Launcher/wallpapers/016_small.jpg"
```

모듈을 **처음** 만든다면, 같은 모듈 폴더에 `module.prop`과 `auto_mount`가 필요하다. 기존에 성공한 모듈 구성을 복제하는 편이 안전하다. SELinux 레이블 역시 원본 JPG와 비교해서 맞춰야 한다.

기존 모듈에 파일을 넣은 뒤 재부팅하여 Magic Mount 적용 여부를 확인한다.

```sh
reboot
```

## 4. 적용 확인

재부팅 후:

```sh
ls -l /system/etc/custom_config/app/Launcher/wallpapers/016*
cmp /system/etc/custom_config/app/Launcher/wallpapers/016.jpg /sbin/.core/img/zte_wallpapers/system/etc/custom_config/app/Launcher/wallpapers/016.jpg
echo $?
```

`cmp` 종료 코드 `0`은 시스템 경로에 표시된 이미지 내용이 모듈 이미지와 같다는 뜻이다. 실제로 **내장 배경화면 선택 화면에 16번째 항목이 나타나는지** 별도 확인한다. 이 기기에서는 확인 완료.

## 결과와 한계

| 실험 | 결과 |
|---|---|
| 기존 내장 배경화면 교체 | 성공 |
| 썸네일 교체 | 성공 |
| 16번째 내장 배경화면 추가 | 성공 |
| APK 재빌드 없이 처리 | 성공 |
| 순정 `/system` 파티션 직접 수정 없이 처리 | 성공 |
| 최대 배경화면 개수 | 미확인 |
| 중간 번호를 건너뛰는 경우 | 미확인 |
| 썸네일이 없는 경우 | 미확인 |

**주의:** 16번째가 표시된다고 해서 무제한 추가가 보장되는 것은 아니다. 구형 Magisk는 현재 마운트된 파일을 삭제·덮어쓸 때 공간 반환 문제가 생길 수 있으므로, 원본과 정상 작동하는 모듈을 백업하고 재부팅 후 확인한다.
