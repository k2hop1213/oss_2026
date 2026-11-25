# Week 12-A. Threads & Synchronization (스레드와 동기화)

## 1. 예제 상황

로그 파일 여러 개(a.log, b.log)를 **동시에** 분석해서 에러 개수를 합친 보고서를 만듦. 파일끼리는 서로 상관없음 → 동시에 처리하기 좋은 일

## 2. 스레드

**프로세스 안의 실행 흐름**

| 스레드마다 따로 | 같은 프로세스끼리 공유 |
|---|---|
| PC(실행 위치), SP와 **스택**(지역 변수) | **코드, 전역 데이터, 힙** |

| | 프로세스 (10주차) | 스레드 (12주차) |
|---|---|---|
| 만들기 | `fork` | `pthread_create` |
| 기다리기 | `waitpid` | `pthread_join` |
| 메모리 | **따로** | **공유** |
| 실패 반환 | -1 | **에러 번호** (0이 성공) |

- **worker** = 작업을 맡은 스레드. main이 준비 → worker들이 분석 → main이 join 후 합침

### 동시성 vs 병렬성

| | 뜻 |
|---|---|
| 동시성 (concurrency) | 실행 기간이 겹침 (코어 1개면 번갈아) |
| 병렬성 (parallelism) | 진짜 같은 순간 (코어 2개 이상) |

## 3. Pthreads

```c
pthread_create(&tid, NULL, worker, &input);   // tid에 ID, NULL 기본 속성, worker 시작 함수, &input 인자
pthread_join(tid, NULL);                      // 그 worker가 끝날 때까지 기다림
```
- 둘 다 **0 = 성공, 아니면 에러 번호** (-1 아님)
- 컴파일: `gcc -pthread ...`

### 기본 예제

```c
typedef struct { const char *path; long errors; int failed; } Job;

static void *scan_file(void *arg) { Job *job = arg; count_errors(job); return NULL; }

int main(void) {
    Job jobs[2] = {{"data/a.log", 0, 0}, {"data/b.log", 0, 0}};
    pthread_t tids[2];
    for (int i = 0; i < 2; i++) check(pthread_create(&tids[i], NULL, scan_file, &jobs[i]));
    for (int i = 0; i < 2; i++) check(pthread_join(tids[i], NULL));
    /* join 후에 결과 출력, total 계산 */
}
```

- 같은 함수, **다른 객체**(jobs[0], jobs[1]) → 각자 자기 칸에 씀
- 주소를 넘김 = 복사가 아니라 원본을 가리킴
- **join 순서가 끝나는 순서를 정하지 않음**. B가 먼저 끝나도 출력은 main 반복문 순서
- join 성공 후에만 결과를 읽음
- 스레드를 쓴다고 **무조건 빨라지지 않음** (생성·대기 비용, 저장소, 코어 부족)

## 4. 완료 ≠ 성공

- join 성공 = 스레드가 **끝났다**는 것만 확인
- 작업이 성공했는지는 **결과(count + 상태)**로 판단

| 경우 | count | 상태 | 의미 |
|---|---|---|---|
| 빈 파일 | 0 | 성공 | 분석 완료, 에러 없음 |
| 파일 없음 | 0 | **실패** | 분석 못 함 |
| 에러 로그 | > 0 | 성공 | 에러 발견 |

## 5. 수명과 생성 실패

- **인자 수명**: worker는 input이 살아 있는 동안 읽어야 함. 지역 변수 주소를 넘기고 함수가 먼저 return하면 사라진 메모리를 읽음 → **join 전까지 살아 있어야**
- **생성 실패**: B 생성이 실패해도 **A는 이미 돌고 있음** → 성공한 ID를 기억 → **시작된 worker는 join** → 그다음 에러 반환

## 6. 공유 변수와 경쟁

### Lost update

```
total = 0. A는 +1, B는 +2
A: 0 읽고 1 계산 / B: 0 읽고 2 계산 → A가 1 씀 → B가 2로 덮어씀 → 2 (기대 3)
```

| 용어 | 뜻 |
|---|---|
| **data race** | 동기화 없이 같은 메모리에 동시 접근, 하나 이상이 쓰기 → C에서 **정의되지 않은 동작** |
| **race condition** | 실행 순서에 따라 결과가 달라지는 논리 오류 |

- **한 번 맞게 나와도 안전하다는 증거가 아님**
- `sleep`으로 순서를 맞추는 건 해결책이 아님

## 7. Mutex (상호 배제)

한 번에 **한 소유자**만. lock ~ unlock 사이 = **임계 구역(critical section)**

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
long total = 0;

/* worker 안 */
pthread_mutex_lock(&mutex);
total += value;              // 읽기 → 더하기 → 쓰기 전부 lock 안
pthread_mutex_unlock(&mutex);
```

| 함수 | 역할 |
|---|---|
| `PTHREAD_MUTEX_INITIALIZER` | 초기화 |
| `pthread_mutex_lock` / `unlock` | 획득 / 해제 |
| `pthread_mutex_destroy` | 모두 끝나고 잠기지 않았을 때만 |

- **읽기도 lock 안에서** (lock 전에 읽으면 옛 값)
- 충돌하는 모든 접근이 **같은 mutex**
- 실행 중엔 읽는 쪽도 mutex, 모든 join 후엔 그냥 읽어도 됨 (mutex는 순서, join은 완료)
- A 먼저: 0→1→3 / B 먼저: 0→2→3 → 합계는 같음

### 결과 합치는 두 방법

| 방법 | mutex |
|---|---|
| 각자 결과 칸 → **join 후 main이 합침** | 필요 없음 |
| worker가 공유 total에 직접 더함 | **같은 mutex 필요** |

## 8. 데드락

| 종류 | 상황 | 해결 |
|---|---|---|
| lock 순서 | A: X 잡고 Y 대기 / B: Y 잡고 X 대기 | **모두 같은 순서**(X → Y) |
| join | main이 mutex를 쥔 채 join, worker는 그 mutex 대기 | **unlock 후 join** |

```c
/* 데드락 */
pthread_mutex_lock(&mutex);
pthread_join(tid, NULL);       // worker는 mutex를 기다림 → 영원히
pthread_mutex_unlock(&mutex);
```

## 9. 일관성 (계좌 이체)

- 이체 = X에서 빼고 Y에 더함 → 합은 그대로 (**invariant**)
- lock을 두 번으로 나누면 reader가 **빼기만 된 중간 상태**를 봄 (각 접근은 보호됐지만 이체 전체는 아님)
- 해결: 이체 전체를 **lock 한 번 안에서**, reader는 lock 안에서 **두 잔액을 함께 복사(snapshot)**, 출력은 lock 밖

## 10. 성능

- 작업 크기가 다르면 **가장 긴 작업 + 비용**이 전체 시간 → 이미지를 **타일로 나눠** 균형 (너무 작으면 관리 비용)
- 직렬 4초 + 병렬 6초:

| worker | 시간 | 속도 향상 |
|---|---|---|
| 1 | 10초 | |
| 2 | 4 + 3 = 7초 | 10/7 ≈ **1.43배** |
| 4 | 4 + 1.5 = 5.5초 | |
| ∞ | 4초 | 최대 **2.5배** |

## 11. 검증

- 실행 증거(참조 결과 비교, 실패 입력, 반복 실행) + **코드 리뷰**(공유 쓰기 보호, 수명, join 후 출력)
- 통과한 실행은 **관찰된 한 순서**만 확인
- 진행 상황은 mutex로 읽고, **최종 결과는 join 후**

## Week 12-A 핵심 요약

- 스레드는 메모리 공유, pthread 함수는 0이 성공
- join 후 결과 읽기, 성공한 스레드만 join
- 공유 갱신은 같은 mutex 안에서 읽기~쓰기 전부
- 데드락: 같은 lock 순서, unlock 후 join
- 각자 칸에 쓰면 mutex 불필요, join은 필요

---

# Week 12-B. Thread Lab

## 준비

```bash
git clone 저장소주소 oss-w12-threads && cd oss-w12-threads
make examples && make demo
```

| 파일 | 역할 |
|---|---|
| `src/main.c` | Job 준비, 결과 보고. **Summary 하나**를 모든 Job이 공유 |
| `src/support.c` | 파일 읽고 지역 개수 세기 (`scan_file`) |
| `src/launch.c` | **활동 1** |
| `src/merge.c` | **활동 2** |
| `activities/seat/seat.c` | **활동 3** |
| `activities/image/strips.c` | **활동 5** |
| `analysis.md` | 예측과 관찰 기록 (**활동 4**) |

| 타깃 | 내 소스 | 제공 |
|---|---|---|
| `make check1` | launch.c | merge.o |
| `make check2` | merge.c | launch.o |
| `make check` | 둘 다 | 공통 지원 소스만 (7/7) |

## 활동 1: launch.c (생성과 join)

```c
int run_jobs(Job jobs[], size_t count) {
    pthread_t tids[16];
    size_t started = 0;
    int rc = 0;
    if (count > 16) return EINVAL;

    for (size_t i = 0; i < count; i++) {                 // TODO 1
        rc = pthread_create(&tids[i], NULL, scan_file, &jobs[i]);
        if (rc != 0) break;                              // 실패하면 멈춤
        started++;                                       // 성공한 것만 셈
    }
    for (size_t i = 0; i < started; i++)                 // TODO 2
        require_ok(pthread_join(tids[i], NULL), "pthread_join");
    return rc;                                           // 전부 성공 0, 아니면 에러 번호
}
```

| 상황 | 답 |
|---|---|
| count=3, 생성 2번 성공 | tids[0], tids[1]만 유효 |
| 3번째 실패 | join **2번** |
| worker가 아직 jobs[0] 사용 중 | **join 끝난 뒤** return |
| 전부 성공 | 0 |

- 결과: `TOTAL lines=7 errors=3 failed=0`, 7/7
- 데모 `/tmp/observe order`: A가 먼저 시작해도 B가 먼저 끝날 수 있음 → 시작 순서 ≠ 끝나는 순서

## 활동 2: merge.c (공유 합계 보호)

```c
void merge_counts(Summary *summary, const Counts *part) {
    require_ok(pthread_mutex_lock(&summary->mutex), "pthread_mutex_lock");
    summary->counts.lines  += part->lines;
    summary->counts.errors += part->errors;
    summary->counts.failed += part->failed;
    require_ok(pthread_mutex_unlock(&summary->mutex), "pthread_mutex_unlock");
}
```
- part(파일 하나의 결과)는 **lock 전에** 이미 계산됨
- **세 값 모두 lock 한 번 안에서** → 반쪽 상태가 안 보임
- `/tmp/observe lost` → total 1 (기대 2) / `/tmp/observe mutex` → 2
- `-fsanitize=thread`(ThreadSanitizer): mutex 없는 `total++`에 **data race** 경고

## 활동 3: seat.c (좌석 예약)

```c
int available = seats > 0;     // 첫 확인 (lock 안)
/* unlock → barrier → lock */
if (available) { --seats; booked[id] = 1; }    // 원래: 옛 값으로 결정 → 2명 승인
if (seats > 0) { --seats; booked[id] = 1; }    // 수정: 예약 순간에 현재 상태 재확인
```

- 실패 순서: A 확인(1석) → B 확인(1석) → A 예약(0석) → B가 **옛 확인 결과로** 예약 → 좌석 1개에 2명
- 확인한 결과가 실제 작업 전에 **낡을(stale) 수 있음** → **확인과 실행은 같은 임계 구역 안에서**
- 좌석 0/1/2개 → confirmed 0/1/2, remaining 0

## 활동 4: 코드 리뷰 (analysis.md에만 작성)

### join 위치

```c
for (...) { pthread_create(...); pthread_join(tids[i], NULL); }
```
- 정확성: 결과는 맞음 / **동시성: 없음** (한 번에 worker 하나 = 순차) / 순환 없음 → 데드락 아님
- 수정: create 반복문과 join 반복문 **분리**

### 순환 대기

```c
pthread_mutex_lock(&summary.mutex);
run_jobs(jobs, count);
pthread_mutex_unlock(&summary.mutex);
```
```
main ──(join: 완료 대기)──▶ worker
 ▲                            │
 └──(mutex를 main이 보유)◀────┘  worker는 merge에서 mutex 대기
```
- 그래도 진행: worker의 파일 읽기·세기 (lock 불필요). 멈추는 곳: merge의 lock
- 수정: **mutex 없이 run_jobs 호출** (unlock 후 기다리기)

## 활동 5: strips.c (이미지 띠 나누기)

- 반전 = **255 − 값** (이웃 픽셀 필요 없음) → 겹치지 않는 범위 [시작, 끝)로 나눔

```c
s[k] = (struct Strip){ k*(N/W), (k+1)*(N/W) };   // 원래: N/W 먼저 → 10/3 = 3
s[k] = (struct Strip){ k*N/W, (k+1)*N/W };       // 수정: 곱한 뒤 나눔
```

| (N, W) | 원래 | 수정 |
|---|---|---|
| (10, 3) | [0,3) [3,6) [6,9) → **9번 누락** | [0,3) [3,6) [6,10) |
| (12, 3) | | [0,4) [4,8) [8,12) |
| (2, 3) | | [0,0) [0,1) [1,2) (빈 띠도 유효) |

- **mutex 불필요**: 서로 다른 칸에만 씀 / **join은 필요**: 다 끝난 뒤 결과를 읽어야 함

## 제출

```bash
cd "$(git rev-parse --show-toplevel)"
make check && git diff
git add src/launch.c src/merge.c analysis.md activities/seat/seat.c activities/image/strips.c
git commit -m "Implement thread activities"
git push
git rev-parse HEAD           # Actions 실행의 SHA와 비교
```
- Actions w12-threads → test → **7/7 checks passed**
- 로컬 통과, Actions 통과, Classroom 제출은 **각각 별개**

## Week 12-B 핵심 요약

- 활동 1: 성공한 것만 join, join 후 return
- 활동 2: lock → 세 값 → unlock
- 활동 3: 확인과 실행을 같은 lock 안에서
- 활동 4: join 반복문 분리, mutex 쥔 채 join 금지
- 활동 5: `k*N/W`, 겹치지 않으면 mutex X, join O

---

# 추가 정리: 복습하며 나온 질문들

## join이 뭐야

그 스레드가 **끝날 때까지 기다리는 것**. 10주차 waitpid와 같은 역할. 끝나면 반환 → 그다음 결과를 읽어도 안전

```
main:    create A ─ create B ─ join A(대기) ─ join B ─ 결과 읽기
worker A:        [분석 → 결과 저장] 끝
worker B:                  [분석 → 결과 저장] 끝
```

## 스레드를 지우는 거야?

아니. 지운 건 TODO 주석과 `return ENOSYS;` 같은 **임시 코드**. 스레드는 create로 만들고, 자기 일(scan_file)을 마치면 스스로 끝나고, join은 그걸 확인·정리

## 이걸 왜 해 (병렬 처리?)

여러 파일을 **동시에** 처리하려는 것. 코어가 여러 개면 실제 병렬, 1개면 번갈아 실행(동시성). 코드는 같고 OS가 정함
