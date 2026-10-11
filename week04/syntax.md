# 4주차 문법: 셸 스크립트

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 기본 구조

```bash
#!/bin/bash
# 또는: #!/usr/bin/env bash
```

실행법: `bash file.sh` / `chmod +x file.sh && ./file.sh`

## 변수

```bash
var=값              # 선언 (= 양옆 공백 없이)
$var                # 값 읽기
"$var"              # 값 읽기 (안전, 공백 보존)
'$var'              # 그대로 문자 "$var" (확장 안 됨)
${var}              # 중괄호로 명확히 구분
$(명령어)            # 명령어 치환 (결과를 값으로)
$((수식))            # 산술 연산
```

## 특수 변수

```bash
$0      스크립트 이름
$1 $2…  위치 인자
$#      인자 개수
$@      모든 인자
$?      직전 명령의 종료 상태 (바로 다음 줄에서 캡처해야 함)
```

## if문

```bash
if [ 조건 ]; then
  ...
elif [ 조건 ]; then
  ...
else
  ...
fi
```

### 숫자 비교

```
-eq  같다       -ne  다르다
-lt  작다       -le  작거나 같다
-gt  크다       -ge  크거나 같다
```

### 문자열 비교

```
=    같다        !=   다르다
-z   빈 문자열    -n   빈 문자열 아님
```

### 파일 테스트

```
-f   일반 파일 존재    -d   디렉토리 존재
-e   존재함            -r   읽기 가능
-w   쓰기 가능         -x   실행 가능
-s   크기 0 아님
```

### 결합

```bash
[ A ] && [ B ]   # AND
[ A ] || [ B ]   # OR
! [ A ]          # NOT
```

## 명령 연결

```bash
cmd1 ; cmd2     # 순서대로 실행
cmd1 && cmd2    # cmd1 성공해야 cmd2 실행
cmd1 || cmd2    # cmd1 실패해야 cmd2 실행
```

## 반복문

```bash
for var in 목록; do
  ...
done

while [ 조건 ]; do
  ...
done

until [ 조건 ]; do
  ...
done
```

`continue` / `break` 루프 제어에 사용

## 함수

```bash
함수이름() {
  local var=값     # 지역변수
  echo "출력값"     # 값을 "리턴"하는 방법 (stdout으로)
  return 0          # 종료 상태 리턴 (값 리턴 아님!)
}

결과=$(함수이름 인자)   # 함수의 stdout을 변수에 저장
```

## 출력 / 에러

```bash
echo "메시지"            # 표준출력
echo "에러" >&2           # 표준에러로 보냄
```

## 디버깅

```bash
bash -n file.sh     # 문법 검사만 (실행 안 함)
bash -x file.sh     # 실행 과정 추적 출력
```
