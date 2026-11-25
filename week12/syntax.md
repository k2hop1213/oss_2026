# 12주차 문법: 스레드 (Pthreads)

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 헤더 / 컴파일

```c
#include <pthread.h>
```

```bash
gcc -pthread file.c -o prog
gcc -O1 -g -pthread -fsanitize=thread -no-pie file.c -o race   # data race 검사
```

## 시작 함수 모양

```c
static void *worker(void *arg) {
    Job *job = arg;
    /* ... */
    return NULL;
}
```

## 생성 / join

```c
pthread_t tid;
int rc = pthread_create(&tid, NULL, worker, &input);   // 0 성공, 아니면 에러 번호
rc = pthread_join(tid, NULL);
if (rc != 0) fprintf(stderr, "%s\n", strerror(rc));
```

## 여러 개: 생성 반복 / join 반복 분리

```c
size_t started = 0;
for (size_t i = 0; i < count; i++) {
    rc = pthread_create(&tids[i], NULL, scan_file, &jobs[i]);
    if (rc != 0) break;
    started++;
}
for (size_t i = 0; i < started; i++)
    pthread_join(tids[i], NULL);
return rc;
```

## Mutex

```c
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&m);
total += value;
pthread_mutex_unlock(&m);

pthread_mutex_destroy(&m);      // 모두 끝나고, 잠기지 않았을 때
```

```c
typedef struct { pthread_mutex_t mutex; Counts counts; } Summary;
Summary s = { .mutex = PTHREAD_MUTEX_INITIALIZER };
pthread_mutex_lock(&s.mutex);
s.counts.lines  += part->lines;
s.counts.errors += part->errors;
s.counts.failed += part->failed;
pthread_mutex_unlock(&s.mutex);
```

## 확인 + 실행을 한 임계 구역에

```c
pthread_mutex_lock(&m);
if (seats > 0) { --seats; booked[id] = 1; }
pthread_mutex_unlock(&m);
```

## 이체 / 스냅샷

```c
pthread_mutex_lock(&m);
x -= amount; y += amount;          // 한 번에
pthread_mutex_unlock(&m);

pthread_mutex_lock(&m);
long cx = x, cy = y;               // 함께 복사
pthread_mutex_unlock(&m);
printf("%ld %ld\n", cx, cy);       // 출력은 lock 밖
```

## 데드락 피하기

```
lock 순서 통일:  모두 X → Y
join 전 unlock:  lock → (작업) → unlock → join
```

## 범위 나누기 [start, end)

```c
s[k] = (struct Strip){ k * N / W, (k + 1) * N / W };
for (size_t i = s.start; i < s.end; i++) img[i] = 255 - img[i];
```

## 속도 향상 한계

```
시간 = 직렬 + 병렬 / worker 수
최대 속도 향상 = 전체 / 직렬      (예: 10 / 4 = 2.5배)
```

## 실습 명령

```bash
make examples && make demo
make check1 && make check2 && make check
gcc -pthread activities/seat/seat.c -o /tmp/seat && /tmp/seat
cd "$(git rev-parse --show-toplevel)/activities/image"
gcc -pthread strips.c -o /tmp/strips && /tmp/strips
```

## 제출

```bash
cd "$(git rev-parse --show-toplevel)"
make check && git diff
git add src/launch.c src/merge.c analysis.md activities/seat/seat.c activities/image/strips.c
git commit -m "Implement thread activities"
git push
git rev-parse HEAD
```
