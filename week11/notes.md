# Week 11-A. Signals (시그널)

## 1. 시그널이란

OS가 프로세스에게 **"이런 사건이 생겼다"**고 알려주는 것. 예: 입력을 읽던 프로세스에 "그만해" 요청

```
생성(generation) → 대기(pending) → 전달(delivery) → 대응 방식(disposition) 적용
```

| 대응 방식 | 뜻 |
|---|---|
| **Default** | 미리 정해진 동작 (대부분 종료) |
| **Ignore** | 버림 (`SIG_IGN`) |
| **Catch** | 등록한 **핸들러 함수** 실행 → 끝나면 원래 코드로 복귀 (같은 프로세스 안) |

### 시그널 출처

| 출처 | 시그널 |
|---|---|
| 터미널 Ctrl+C | **SIGINT** |
| 다른 프로세스 (`kill`) | **SIGTERM** |
| 자식 프로세스 상태 변화 | **SIGCHLD** |

## 2. 시그널 이름과 기본 동작

| 시그널 | 기본 동작 | 용도 |
|---|---|---|
| SIGINT / SIGTERM | 종료 | Ctrl+C / 종료 요청 |
| SIGTSTP | 정지 | Ctrl+Z |
| SIGSTOP / SIGCONT | 정지 / 재개 | |
| SIGKILL | 종료 | 강제 종료 |
| SIGUSR1 | 종료 | 앱이 정하는 요청 |
| SIGALRM | 종료 | `alarm` 타이머 |

> ⚠️ **SIGKILL, SIGSTOP은 핸들러 등록·차단·무시 모두 불가**

## 3. 작업 제어 (터미널)

| | 뜻 |
|---|---|
| 포그라운드 | 터미널 입력을 가진 작업. **Ctrl+C, Ctrl+Z가 여기로** |
| 백그라운드 | 입력은 Bash가 받고 작업은 뒤에서 계속 |

```bash
sleep 60        # Ctrl+Z → [1]+ Stopped
jobs            # 이 셸의 작업 목록
bg %1           # 1번을 백그라운드에서 재개 → sleep 60 &, Running
fg %1           # 앞으로 → Ctrl+C로 종료
sleep 60 &      # 처음부터 백그라운드
echo $!         # 마지막 백그라운드 PID
```

- **포그라운드/백그라운드(입력)와 실행/정지(상태)는 별개** → "백그라운드인데 정지된 작업"도 있음
- Running → (SIGTSTP/SIGSTOP) → Stopped → (SIGCONT) → Running

## 4. kill: PID로 시그널 보내기

```c
if (kill(pid, SIGTERM) == -1) perror("kill");   // 0 = 요청 보냄, -1 = 에러
```
- 이름과 달리 "보내기" 함수. 양수 pid = 그 프로세스 하나

## 5. 핸들러로 정리 후 종료 (stop_flag.c)

```c
static volatile sig_atomic_t stop = 0;

static void request_stop(int signo) { (void)signo; stop = 1; }

int main(void) {
    struct sigaction sa = {0};
    sa.sa_handler = request_stop;
    sigemptyset(&sa.sa_mask);
    if (sigaction(SIGTERM, &sa, NULL) == -1) return 1;

    printf("READY pid=%ld\n", (long)getpid()); fflush(stdout);
    while (!stop) nanosleep(&(struct timespec){0, 100000000}, NULL);
    puts("Cleanup in main");
    return 0;
}
```

| 부분 | 뜻 |
|---|---|
| `sig_atomic_t` | 한 번에 읽고 쓰는 정수형 |
| `volatile` | 컴파일러가 접근을 생략하지 않게 |
| `sa_handler` | 부를 함수 |
| `sa_mask` 비움 | 핸들러 중 추가로 막을 시그널 없음 |
| `sigaction(..., NULL)` | 등록 (실패 -1), 이전 설정 저장 안 함 |

```bash
./bin/stop-flag & pid=$!
kill -TERM "$pid"          # → Cleanup in main
wait "$pid"; echo $?       # 0
```

### 비동기 시그널 안전성 (async-signal-safe)

- 핸들러는 main이 stdio 버퍼를 고치는 **도중**에도 끼어들 수 있음
- **printf, fopen, malloc은 핸들러에서 안전하지 않음** → 핸들러는 **플래그만**, 처리는 main

## 6. 요청 손실과 차단

```c
int requested = snapshot_requested;   // 0 복사
// ← 여기서 핸들러가 snapshot_requested = 1
snapshot_requested = 0;               // 새 요청이 지워짐 → 요청 손실
```

| | 뜻 |
|---|---|
| **차단 (blocked)** | 전달을 **미룸** → pending → 차단 풀면 전달 |
| **무시 (ignored)** | 버림, 핸들러 안 돌아감 |

```c
sigset_t set, old;
sigemptyset(&set); sigaddset(&set, SIGUSR1);
sigprocmask(SIG_BLOCK, &set, &old);       // 차단
int requested = snapshot_requested;       // 복사 + 지우기
snapshot_requested = 0;
sigprocmask(SIG_SETMASK, &old, NULL);     // 복원 → 그때 pending 전달
```

- **Coalescing**: 차단 중 SIGUSR1을 3번 보내도 pending은 **1개** → 시그널로 **개수를 세면 안 됨**
- 시그널은 "확인할 일이 있다"는 알림, 개별 작업은 앱 데이터에 따로 저장

## 7. 자식 회수 (SIGCHLD)

- 자식이 끝나면 부모에 **SIGCHLD** → 부모가 waitpid로 회수
- **좀비(zombie)**: 종료됐지만 아직 회수 안 된 자식

```c
for (;;) {
    pid_t pid = waitpid(-1, &status, WNOHANG);   // -1 아무 자식, WNOHANG 안 기다림
    if (pid > 0) continue;                       // 하나 회수
    if (pid == -1 && errno == EINTR) continue;
    break;
}
```

| 결과 | 의미 |
|---|---|
| 자식 PID | 그 자식 회수 |
| 0 | 실행 중인 자식만 남음 |
| -1, ECHILD | 회수할 자식 없음 |

## 8. 보내기와 완료

```c
if (kill(pid, SIGTERM) == -1) { perror("kill"); return 1; }
while (waitpid(pid, &status, 0) == -1)
    if (errno != EINTR) { perror("waitpid"); return 1; }
```

- **kill = 전달 요청일 뿐, 완료의 증거는 waitpid**

| | 결과 |
|---|---|
| SIGTERM + 핸들러 | 정리 코드 도달 → `WIFEXITED`, **exit=0** |
| SIGKILL | 정리 건너뜀 → `WIFSIGNALED`, **signal=9** |

## Week 11-A 핵심 요약

- SIGINT=Ctrl+C, SIGTSTP=Ctrl+Z, SIGTERM=kill, SIGKILL/SIGSTOP은 못 막음
- 핸들러는 `volatile sig_atomic_t` 플래그만, 처리는 main
- 플래그 읽고 지우는 구간은 sigprocmask로 차단 → 요청 보존. 개수 세기 X
- kill 후 waitpid까지 해야 완료. SIGTERM은 정리 가능, SIGKILL은 불가

---

# Week 11-B. Signal Lab (시간 제한 실행기 limit)

## 준비

```bash
git clone 저장소주소 oss-w11-signals && cd oss-w11-signals
make demo
command -v ping tail timeout           # 없으면 sudo apt-get install -y iputils-ping
mkdir -p out; cp data/events.log out/live.log
```

## 터미널 시그널 관찰

```bash
ping 127.0.0.1                         # Ctrl+C → SIGINT → 통계 출력 후 종료 (핸들러 사용 예)
tail -n 1 -f out/live.log              # -n 1 마지막 줄, -f 계속 읽기
^Z                                      # SIGTSTP → [1]+ Stopped
jobs
printf 'ERROR request=/new status=500\n' >> out/live.log
bg                                      # SIGCONT → 재개 → 이제 ERROR 줄이 보임
fg                                      # 앞으로
^C                                      # 종료
```
- 새 줄이 bg 후에야 보이는 이유: **정지 중엔 파일을 못 읽음**, SIGCONT로 재개돼야 읽어서 출력

```bash
timeout -v 2s tail -n 1 -f data/events.log
echo $?                                 # 124 (시간 초과) — 시그널을 보낸 건 timeout 프로그램
```

## limit 구조

```
부모(limit): fork → alarm(초) → monitor_child(감시) → 시간 되면 SIGALRM
            → kill(child, SIGTERM) → waitpid 회수 → 결과 반환
자식: execvp(argv[2], &argv[2]) (실패 127)
```

```bash
./bin/limit-demo 2 tail -n 1 -f data/events.log
# START parent=577 child=578 limit=2s
# REQUEST child=578 signal=SIGTERM
# COLLECTED child=578 status=143        ← 128 + 15 (SIGTERM)
# $? = 124
```
- **시그널을 보낸 뒤에도 회수 책임은 부모** (REQUEST 다음 COLLECTED)

| 상황 | limit 종료 코드 |
|---|---|
| 시간 초과 (SIGALRM) | **124** |
| Ctrl+C / kill -INT 부모 (SIGINT) | **130** |
| exec 실패 | **127** |
| 그 외 | 자식 결과 그대로 |

## 활동 1: control.c

```c
volatile sig_atomic_t requested_signal = 0;

static void on_signal(int signo)
{
    requested_signal = signo;                          // TODO 1
}

int install_control(void)
{
    struct sigaction sa = {0};
    sa.sa_handler = on_signal;
    sigemptyset(&sa.sa_mask);
    if (sigaction(SIGALRM, &sa, NULL) == -1) return -1;   // TODO 2
    if (sigaction(SIGINT,  &sa, NULL) == -1) return -1;
    return 0;
}
```
- 두 시그널 모두 같은 핸들러, 어떤 것인지는 signo로 구분 (SIGALRM 14, SIGINT 2)
- `make check1`

## 활동 2: monitor.c

```c
for (;;) {
    pid_t result = waitpid(child, status, WNOHANG);
    if (result == child) return 0;                              // TODO 1
    if (result == -1 && errno != EINTR) return -1;
    if (requested_signal && !sent) {                            // TODO 2
        if (kill(child, SIGTERM) == -1 && errno != ESRCH) return -1;
        sent = 1;
    }
    nanosleep(&(struct timespec){0, 100000000}, NULL);
}
```
- WNOHANG 결과: child PID → 회수 / 0 → 실행 중 / -1 → errno (EINTR 재시도)
- SIGTERM은 **한 번만**, 보낸 뒤에도 **회수될 때까지 반복**
- ESRCH = 대상 프로세스가 이미 없음
- `make check2` → `make check` (6/6)

## 활동 3: 예측

```bash
./bin/limit 5 true ; echo $?                          # REQUEST 없음, 0
./bin/limit 5 grep ERROR data/events.log ; echo $?    # 0 (찾음)
./bin/limit 5 grep FATAL data/events.log ; echo $?    # 1 (못 찾음, 시간 초과 아님)
./bin/limit 2 tail -n 1 -f data/events.log ; echo $?  # REQUEST, 124
```
- grep은 **스스로 끝나며** 1 → limit은 그대로 전달 / tail은 안 끝나서 **부모가 시간 초과로 TERM** → 124

```bash
./bin/limit 30 tail -f data/events.log &
parent=$!
kill -INT "$parent"          # 부모만 SIGINT → 부모가 자식에게 TERM
wait "$parent"; echo $?      # 130
```

### 잘못된 코드 리뷰

```c
if (requested_signal && !sent) {
    kill(child, SIGTERM);
    return 0;               /* 자식을 회수하지 않음 */
}
```
- 빠진 것: kill 반환값 검사, **waitpid 회수 전에 return 0**
- return 0은 **waitpid가 child PID를 반환했을 때만**
- 영향: status가 안 채워졌는데 main이 해석 → 잘못된 결과, 자식은 좀비로 남거나 계속 실행

## 제출

```bash
make check
git add src/control.c src/monitor.c
git commit -m "Implement signal handling"
git push                     # Actions command-limit → test → 6/6 passed
```

## Week 11-B 핵심 요약

- control: `requested_signal = signo` + SIGALRM·SIGINT 등록
- monitor: WNOHANG waitpid 반복, TERM 한 번, 회수 후에만 return 0
- 124 시간 초과 / 130 SIGINT / 127 exec 실패 / 그 외 그대로

---

# 추가 정리: 복습하며 나온 질문들

## 포그라운드 / 백그라운드

기준은 **지금 키보드 입력을 누가 받나**

| 단계 | tail 상태 | 입력 받는 쪽 |
|---|---|---|
| `tail -f ...` | 포그라운드 + 실행 | tail |
| Ctrl+Z | 정지 | 셸 (`$` 나옴) |
| `bg` | 백그라운드 + 실행 | 셸 |
| `fg` | 포그라운드 + 실행 | tail |
| Ctrl+C | 종료 | 셸 |

비유: 포그라운드 = 보고 있는 창, 백그라운드 = 최소화한 창, Stopped = 일시정지된 창

## bg / fg / jobs

| 명령 | 하는 일 |
|---|---|
| `jobs` | 작업 목록과 상태를 **보여줌** |
| `bg` | 정지된 작업을 **뒤에서** 다시 실행 |
| `fg` | 작업을 **앞으로** 가져옴 (정지돼 있으면 실행도) |

- 번호 없으면 `+`(가장 최근) 작업에 적용. `%1`처럼 지정 가능
- 마지막에 `fg`를 하는 이유: **Ctrl+C는 포그라운드 작업에만 감**

## jobs 출력 읽기

```
[1]   Stopped   tail -n 1 -f out/live.log
[3]+  Stopped   tail -n 1 -f out/live.log
[4]   Running   tail -n 1 -f out/live.log &
```
- `[n]` 작업 번호 / `+` 최근, `-` 그 전 / 상태 / 명령
- 끝의 `&`는 직접 친 게 아니어도 **bg로 재개하면 bash가 붙여서 표시**
- jobs에 올라가는 것: 이 셸에서 실행했는데 **아직 안 끝난** 작업 (백그라운드 실행, Ctrl+Z로 정지). 다 끝난 명령이나 다른 터미널 작업은 안 보임
- tail -f는 스스로 안 끝나서 실행하고 Ctrl+Z할 때마다 쌓임 → `kill %1 %2 %3 %4`로 정리
