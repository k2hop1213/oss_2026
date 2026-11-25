# 시험 대비 복습 포인트 (연습 문제에서 틀린 것)

## 🔴 꼭 복습

**1. `$?`는 바로 직전 명령 하나의 상태** (W04A 24~25쪽)

```bash
grep -q ERROR clean.log   # 1
echo "checked"            # 0 ← $?가 echo 것으로 덮어써짐
echo $?                   # 0
```

grep 결과가 필요하면 바로 다음 줄에서 `status=$?`로 저장한다. `grep -q`는 출력 없이 상태만 남긴다.

**2. 비교 연산자 경계값**

| `-lt` | `-le` | `-gt` | `-ge` | `-eq` | `-ne` |
|---|---|---|---|---|---|
| < | ≤ | > | ≥ | = | ≠ |

끝이 **e**면 같음 포함. `5 -lt 5`는 거짓. `&&`는 둘 다 참이어야 참.

**3. 오버플로 계산:** 255 초과 → −256, 0 미만 → +256. 검산 필수.

**4. `git add` 후 또 수정 (`MM`)**
- `add`는 그 순간 내용을 사진 찍듯 복사한다
- commit하면 **add한 시점의 버전**만 들어감
- `git diff` = add 안 한 수정 / `git diff --staged` = 커밋될 수정

## 🔵 9~12주차 헷갈렸던 것

**1. getline으로 읽은 줄에는 `\n`이 붙어 있다**
- 헤더 비교 전에 `line[strcspn(line, "\r\n")] = '\0';`

**2. `write report: No such file or directory` = 출력 폴더가 없는 것**
- `O_CREAT`는 파일만 만들고 폴더는 안 만듦 → `mkdir -p output`

**3. fdopen의 fd는 바로 위 open이 준 번호, fclose가 그 fd까지 닫는다**
- fclose 후 close(fd)를 또 하면 이중 닫기

**4. jobs의 `&`는 직접 안 쳐도 bg로 재개하면 붙는다**
- `&` = 지금 백그라운드에서 실행 중

**5. join은 스레드를 지우는 게 아니라 끝날 때까지 기다리는 것**
- 지운 건 TODO 주석과 `return ENOSYS;` 같은 임시 코드

## 🟡 답안 작성 습관

- 명령어는 끝까지: `-Iinclude`(`-` 필수), `git restore --staged 파일이름`
- "~로 설명하시오" → 그 용어(T/U 등)를 답에 넣기
- 에러 이름 정확히: `undefined reference to '함수'`
- 출력 예측은 화면에 나오는 줄 전부, 줄바꿈까지

## 🟢 이미 잘하는 것

파이프와 리다이렉션, `sort` vs `sort -n`, `$@ $< $^`, Make 의존성 계산, T/U로 링크 에러 구분
