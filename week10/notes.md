# Week 10-A. Processes (프로세스와 프로그램 실행)

## 1. 이번 주 목표: 명령 실행기(command runner)

다른 프로그램을 실행하고 결과(종료 코드)를 받는 프로그램을 만들며 **fork → exec → wait**를 배웁니다. 셸, make, CI가 하는 일과 같습니다.

```
부모: fork() ─────────────▶ waitpid() 기다림 → 종료 상태 받음 → 계속
자식:   └─ execvp() → hello 실행 → exit(0)
```

## 2. 프로그램 vs 프로세스

| | 뜻 |
|---|---|
| 프로그램 | 저장된 코드와 데이터 (`bin/hello`) |
| 프로세스 | **실행 중인 인스턴스**. PID, 메모리, 실행 위치를 가짐 |

- 같은 프로그램으로 프로세스 여러 개 → 각각 **자기 메모리** (4101은 value=10, 4102는 value=20)
- 프로세스가 가진 것: 메모리(코드, 전역 데이터, 힙, 스택), 실행 위치(instruction pointer, 레지스터), 열린 fd(0, 1, 2)
- 크롬: 렌더러를 프로세스로 나눔 → 렌더러 하나가 죽어도 **그 프로세스가 맡은 페이지만** 영향 (메모리 분리)
- **부모/자식**: 만든 쪽 / 만들어진 쪽. **PPID** = 부모의 PID

## 3. fork()

```c
pid_t pid = fork();
if (pid < 0)       { /* 실패: 부모만 있음 */ }
else if (pid == 0) { /* 자식 */ }
else               { /* 부모: pid = 자식 PID */ }
```

| 반환 | 누가 받나 |
|---|---|
| 자식 PID (> 0) | 부모 |
| **0** | 자식 |
| **-1** | 실패 (부모) |

- fork 한 번 → **두 프로세스 모두 fork 다음 줄부터** 실행, 반환값만 다름
- **누가 먼저 실행될지는 OS가 정함** → 출력 순서가 바뀔 수 있음
- 순서가 필요하면 `waitpid(pid, NULL, 0)` → 출력이 항상 child, parent

### fork 후 메모리는 따로

```
value = 10 → fork → 자식이 value = 20
child  pid=537 value=20
parent pid=320 child=537 value=10      ← 부모는 그대로
```
- 12주차 스레드(메모리 공유)와 비교 포인트

## 4. execvp()

- 성공하면 지금 프로세스의 코드·메모리가 **새 프로그램으로 바뀜**, **PID는 그대로**
- 성공하면 **돌아오지 않음** → 다음 줄은 실패했을 때만 실행

```c
if (pid == 0) {
    execvp(argv[0], argv);
    perror(argv[0]);      // 여기 왔다 = exec 실패
    _exit(127);           // 명령 실행 못 함 (셸 command not found와 같은 값)
}
```

- `_exit`: 부모에게서 복사된 stdio 버퍼를 다시 비우지 않고 바로 종료
- argv 원소 하나 = 인자 하나, 끝은 **NULL**

```c
char *argv[] = {"gcc", "sample file.c", "-o", "bin/hello", NULL};
// 셸: gcc "sample file.c" -o bin/hello
```

## 5. waitpid()와 종료 상태

```c
int status;
pid_t finished = waitpid(pid, &status, 0);
if (finished < 0) { perror("waitpid"); return 125; }
```

| 매크로 | 뜻 |
|---|---|
| `WIFEXITED(status)` | 정상 종료? |
| `WEXITSTATUS(status)` | 종료 코드 |
| `WIFSIGNALED(status)` | 시그널로 종료? |
| `WTERMSIG(status)` | 시그널 번호 |

- 자식이 7 반환 → status = **1792** → WIFEXITED 참 → WEXITSTATUS = **7**
- status는 정보가 합쳐진 숫자라 **매크로로 꺼내야** 함
- Ctrl+C → SIGINT → `sleep 60` 종료 → WIFSIGNALED

## 6. runner 전체 흐름

```c
int run_one(char *const argv[]) {
    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 125; }
    if (pid == 0) { execvp(argv[0], argv); perror(argv[0]); _exit(127); }
    int status;
    while (waitpid(pid, &status, 0) < 0) {
        if (errno == EINTR) continue;        // 시그널로 끊김 → 다시 기다림
        perror("waitpid"); return 125;
    }
    return decode_status(status);
}
```

- main: `run_one(&argv[1])` (argv[0]인 runner 자신을 빼고 넘김) → `runner: exit=%d`를 stderr에 → 그 코드를 그대로 반환
- 컴파일 결과로 실행 여부 결정: `if (run_one(build) != 0) return code;` → 성공했을 때만 실행 (make, CI와 같음)

| 상황 | 결과 |
|---|---|
| 있는 명령 | 0 유지 |
| 없는 명령 (exec 실패) | **127** |
| 7 반환 프로그램 | **7** (그대로 전달) |
| runner 자체 오류 (fork/wait 실패) | **125** |

- 도구 7 → decode_status 7 → main 7 → 셸 `$?` = 7
- 자식은 부모의 **열린 fd를 물려받음** → 자식 출력과 runner 메시지가 같은 터미널에

## Week 10-A 핵심 요약

- fork: 부모 = 자식 PID / 자식 = 0 / 실패 = -1. 메모리는 따로
- execvp: 성공하면 안 돌아옴, PID 유지. 실패면 perror + `_exit(127)`
- waitpid + WIFEXITED/WEXITSTATUS. EINTR이면 다시 기다림
- runner는 결과를 바꾸지 않고 전달

---

# Week 10-B. Command Shell Lab (작은 셸 만들기)

## 준비

```bash
git clone 저장소주소 oss-w10-submission && cd oss-w10-submission
make demo && ./bin/shell-demo
oss-shell> wc -l data/sample.csv      # 6 data/sample.csv  [exit=0]
oss-shell> ./missing-command          # No such file or directory  [exit=127]
oss-shell> quit
```

- 셸은 명령을 실행하고 **다음 입력을 다시 받아야** 함 → fork로 셸을 보존

| 파일 | 역할 |
|---|---|
| `provided/main.c` | 명령 반복문 (프롬프트 → read_command → builtin? → start → wait) |
| `provided/input.c` | fgets + strtok으로 NULL로 끝나는 argv |
| `src/start.c` | **TODO S**: 명령 시작 |
| `src/wait.c` | **TODO W**: 결과 읽기 |
| `src/builtin.c` | **TODO C**: cd |

```
make start / make status / make builtin    # 내 파일 하나 + 나머지는 libsupport.a
make check                                  # 셋 합쳐 bin/oss-shell, 9개 검사
```

## 활동 1: start.c

```c
pid_t start_command(char *const argv[]) {
    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return -1; }
    if (pid == 0) { execvp(argv[0], argv); perror(argv[0]); _exit(127); }
    return pid;                 // 기다리는 건 main
}
```

- 검사 `make check-start`: 인자 전달, 없는 명령 127, **셸이 다음 명령을 받음**
- **fork 없이 execvp만 하면?** wc 출력은 똑같이 나오지만 **셸 자신이 wc로 바뀌어** wc가 끝나면 사라짐 → 다음 `/bin/echo NEXT`를 oss-shell이 못 받음

## 활동 2: wait.c

```c
/* TODO W: 시그널 분기 앞에 */
if (WIFEXITED(status)) return WEXITSTATUS(status);
```
- 제공: EINTR 재시도, wait 에러 **125**, 시그널 종료 **128 + 시그널 번호**

| probe 7 (status 1792) | 출력 |
|---|---|
| A `return 0;` | [exit=0] |
| B `return status;` | [exit=1792] |
| **C** `WEXITSTATUS(status)` | **[exit=7]** ✅ |

- 0으로 끝나는 명령만 테스트하면 status도 0 → A, B, C 모두 [exit=0] → **버그를 못 찾음**. 0이 아닌 코드로도 테스트

## 활동 3: builtin.c (cd)

```c
/* TODO C */
if (chdir(argv[1]) != 0) perror("cd");
return 1;                       // 처리했음 → main이 fork 없이 다음 입력
```

- 상대 경로는 **호출한 프로세스의 현재 디렉터리** 기준
- 자식에서 chdir → 자식만 이동하고 종료 → **부모 셸은 그대로** → cd는 셸 자신이 처리하는 **builtin**
- 부모에서 chdir → 다음 자식이 그 디렉터리를 물려받음

### 둘 다 통과하는 검사 강화

| 구현 | `cd data → cd .. → wc -l data/sample.csv` | `cd data → pwd` |
|---|---|---|
| A 셸에서 chdir (정답) | 6 | .../data |
| B 자식에서 chdir (버그) | 6 | /work/oss-w10-submission |

→ 기존 검사는 차이를 못 잡음, **cd 다음에 pwd**를 넣으면 구분됨

## 실패 원인 찾기

| 명령 | 결과 | 누가 실패를 보고 |
|---|---|---|
| `grep ERROR data/events.log` | 2줄, exit 0 | |
| `grep WARNING data/events.log` | 출력 없음, exit 1 | grep (못 찾음) |
| `grep ERROR data/missing.log` | exit 2 | **grep** (execvp는 성공) |
| `./missing-command` | exit 127 | **셸 자식의 perror** (execvp 실패) |
| `/bin/echo NEXT` | NEXT, exit 0 | 셸이 살아 있다는 확인 |

- 실패한 cd 테스트: `cd data → cd ./missing-dir → pwd` → 여전히 .../data여야 함 (에러 메시지만 보면 상태 변화를 못 잡음)

## 제출

```bash
make check
git add src && git commit -m "Implement command shell" && git push
```
- Actions → shell-check → check → Build and check. 실패하면 input / expected / actual 확인

## Week 10-B 핵심 요약

- start: fork → 자식 execvp(실패 127) → 부모 pid 반환
- wait: `WIFEXITED → WEXITSTATUS`
- cd: 부모에서 chdir, return 1
- 테스트는 0이 아닌 종료 코드, pwd 같은 **상태**까지 확인

---

# 추가 정리: 복습하며 나온 질문들

## run_one 앞부분이 하는 일

fork로 자식 생성 → 실패면 125 → 자식은 execvp로 명령 실행 → 실패하면 perror + `_exit(127)`. 부모는 그다음 waitpid로 기다리고 종료 코드를 꺼내 반환

## C 전부 알아야 하나

실습 TODO 몇 줄이 핵심. 각 줄이 **없으면 어떻게 되는지** 말할 수 있으면 됨

- `_exit(127)`이 없으면 → exec 실패한 자식이 부모 코드를 계속 실행
- `WEXITSTATUS` 대신 `return status` → 1792 같은 값
- fork 없이 exec → 셸이 사라짐
