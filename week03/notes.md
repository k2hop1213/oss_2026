# Week 03-A. Git Basics

## 1. 왜 Git이 필요한가

**현재 파일만으로는 "어떻게 변해왔는지" 알 수 없다**는 게 핵심 문제.

| 작업 방식 | 현재 파일 | 이전 내용 |
|---|---|---|
| 그냥 편집하고 저장 | 갱신됨 | **덮어써져 사라질 수 있음** |
| 다른 파일로 복사 | 별도 복사본 | 사람이 수동으로 관리해야 함 |
| **Git 커밋** | 작업 파일 그대로 유지 | **저장소에 기록됨** |

**버전 히스토리**: 각 커밋은 **작성자, 메시지, 부모 커밋**을 가진 스냅샷.

예시 (README.md 변경 과정):
- Commit A: 첫 줄 추가 (문서 생성)
- Commit B: Git 설명 추가 (목적 설명)
- Commit C: 리뷰 알림 추가 (작업 규칙 기록)

---

## 2. Git vs GitHub

| | **Git (로컬 컴퓨터)** | **GitHub (서버)** |
|---|---|---|
| 역할 | 파일 편집+커밋 생성, 이전 버전 확인, 오프라인 작업 가능 | 공유 저장소 호스팅, 접근 권한 관리, 협업 리뷰 |
| 핵심 문장 | **"Git이 히스토리를 만든다"** | **"GitHub는 그 히스토리의 공유 사본을 호스팅한다"** |

---

## 3. 실습 환경 준비

```bash
sudo apt-get update
sudo apt-get install git
git --version
```

- `git-practice`라는 새 디렉토리로 실습
- `README.md`라는 평문 텍스트 파일 하나로 실습
- **로컬 예제는 GitHub 계정이 필요 없음**

---

## 4. 저장소 생성

```bash
$ mkdir git-practice && cd git-practice
$ git init -b main
Initialized empty Git repository in /home/student/git-practice/.git/
$ ls -a
.  ..  .git
```

- `git init -b main`: **`main`이라는 이름의 브랜치**로 빈 저장소 생성
- `.git` 폴더가 실제 저장소 데이터가 담기는 곳

---

## 5. 커밋 정체성(Identity) 설정

```bash
$ git config user.name "Student"
$ git config user.email "student@example.com"
```

> ⚠️ 이건 저장소별 로컬 설정일 뿐, **GitHub 로그인과는 무관**함

---

## 6. Git의 세 가지 영역 (핵심 개념)

```
Working Directory  →  Staging Area  →  .git directory (Repository)
   (작업 디렉토리)      (스테이징 영역)      (저장소/히스토리)
```

| 영역 | 의미 |
|---|---|
| **Working tree** | 지금 눈에 보이는 현재 파일들 |
| **Index (Staging area)** | 다음 커밋에 포함될 스냅샷(선택된 상태) |
| **Repository** | 커밋된 히스토리 전체 |

흐름: `Stage Fixes`(작업→스테이징) → `Commit`(스테이징→저장소) → `Checkout`(저장소→작업)

---

## 7. 첫 파일 만들고 상태 확인

```bash
$ echo "Linux practice" > README.md
$ git status --short
?? README.md
```

`??`는 **"추적되지 않는(untracked) 파일"** 표시

| 마지막 커밋 | 스테이징 영역 | 작업 트리 |
|---|---|---|
| 없음 | 아직 선택 안 됨 | 1줄 |

> 파일이 디스크에 존재해도 커밋에 포함되지 않을 수 있음

---

## 8. 스테이징 (git add)

```bash
$ git add README.md
$ git status
Changes to be committed:
        new file:   README.md
```

`git add`는 **스테이징 당시의 파일 내용을 기록**함 (이 시점 스냅샷이 "찜"되는 것)

---

## 9. 첫 커밋 (git commit)

```bash
$ git commit -m "Add practice README"
[main (root-commit) aca6922] Add practice README
$ git status
nothing to commit, working tree clean
```

> ⚠️ "working tree clean"이 곧 "GitHub에 업로드됐다"는 뜻은 아님 — 아직 로컬 저장소에만 있는 상태

---

## 10. 추적 중인 파일 편집

```bash
$ echo "Git records snapshots." >> README.md
$ git status --short
 M README.md
```

| 마지막 커밋 | 스테이징 | 작업 트리 |
|---|---|---|
| 1줄 | 1줄 | 2줄 |

> 파일을 저장한다고 스테이징 영역이 자동으로 갱신되지 않음

---

## 11. git diff 읽기

```bash
$ git diff
diff --git a/README.md b/README.md
--- a/README.md
+++ b/README.md
@@ -1 +1,2 @@
 Linux practice
+Git records snapshots.
```

- `git diff` → **작업 트리 vs 스테이징 영역** 비교
- `git diff --staged` → **스테이징 영역 vs 마지막 커밋** 비교
- `+` 표시는 추가된 줄

---

## 12. add 이후 또 편집 (MM 상태)

```bash
$ echo "Review before commit." >> README.md
$ git status --short
MM README.md
```

`git status --short`는 **두 글자 칸**으로 구성됨: `XY 파일명`
- **X (첫 칸)**: 스테이징 영역 상태
- **Y (둘째 칸)**: 작업 트리 상태

| 표시 | 의미 |
|---|---|
| ` M` (공백+M) | 스테이징 안 됨, 작업 트리만 변경 |
| `M ` (M+공백) | 스테이징 됨, 커밋 대기 중 |
| `MM` | 스테이징도 됐고, **그 이후에 또** 변경됨 |

이때 파일 하나가 **세 가지 버전**으로 동시에 존재:

| 마지막 커밋 | 스테이징 영역 | 작업 트리 |
|---|---|---|
| Linux practice | Linux practice | Linux practice |
| | Git records snapshots. | Git records snapshots. |
| | | Review before commit. |

> 다음 커밋은 가운데(스테이징) 열을 사용함

---

## 13. 스테이징된 버전 커밋

```bash
$ git commit -m "Describe Git snapshots"
$ git show HEAD:README.md
Linux practice
Git records snapshots.
$ cat README.md
Linux practice
Git records snapshots.
Review before commit.
```

> **커밋 후에도 3번째 줄은 작업 파일에 그대로 남아있음** (아직 스테이징 안 됐으니까)

남은 변경분을 다시 add→commit:
```bash
$ git add README.md
$ git commit -m "Add review reminder"
```

---

## 14. 커밋 히스토리 읽기

```bash
$ git log --oneline
1b6acc9 Add review reminder
da60bf0 Describe Git snapshots
aca6922 Add practice README
```

- **최신 커밋이 맨 위**
- 커밋 ID(해시)는 메타데이터에 따라 달라짐

```bash
$ git show aca6922:README.md
Linux practice
```
`git show <커밋ID>:<파일명>`으로 **과거 특정 시점의 파일 내용** 확인 가능 (작업 파일은 안 바뀜)

---

## 15. 변경 되돌리기 (git restore)

### 스테이징 취소 (unstage)
```bash
$ git restore --staged README.md
```
→ **인덱스(스테이징)만 되돌림, 작업 파일 내용은 보존됨**

### 스테이징 안 된 편집 폐기
```bash
$ git restore README.md
```
→ **인덱스에 있는 내용으로 작업 파일을 되돌림** (스테이징 이후의 편집 내용이 사라짐 — 주의!)

---

## 16. .gitignore로 생성 파일 무시하기

```bash
$ echo "*.log" > .gitignore
$ echo "local output" > session.log
$ git status --short
?? .gitignore
$ git status --short --ignored
?? .gitignore
!! session.log
```

- `.gitignore`에 패턴을 적으면 해당 파일들은 `git status`에 안 뜸
- `!!`는 "무시된 파일" 표시
- **`.gitignore`는 추적 안 된(untracked) 파일에만 영향**, 이미 커밋된 히스토리는 안 지움

### 이미 추적 중인 파일을 무시하고 싶다면
`.gitignore`만 적어서는 이미 추적 중인 파일에 효과가 없음. `git rm --cached`로 별도 처리 필요:
```bash
$ echo "*.log" > .gitignore          # 앞으로 추적 안 할 패턴
$ git rm --cached app.log             # 이미 추적 중이던 파일 untrack
$ git commit -m "Stop tracking app.log"
```
> `.gitignore`는 텍스트 파일일 뿐 사람이 직접 써야 바뀌고, `git rm --cached`는 이미 커밋/스테이징된 파일을 앞으로의 추적 대상에서 빼는 것 — 둘은 별개의 작업

---

## 17. 좋은 커밋 메시지

| 애매한 메시지 | 구체적인 메시지 |
|---|---|
| update | Add practice README |
| fix | Correct the output filename |
| done | Ignore generated log files |

> 커밋 전에 스테이징된 diff를 리뷰하는 습관 중요 (AI 도움 받은 편집도 포함)

---

## 18. 커밋과 부모(Parent) 관계

```
A ← B ← C
(parent)  (parent)
```

각 커밋은 **이전 커밋을 부모로 가리키는 화살표**로 연결됨. 시간은 오른쪽으로 진행, 화살표는 과거(부모)를 가리킴.

**디렉토리 트리 vs 커밋 그래프**

| | 설명하는 것 |
|---|---|
| 디렉토리 트리 | 파일들의 **위치** |
| 커밋 그래프 (A→B→C) | **버전 히스토리** |

---

## 19. 브랜치(Branch)

```
        C  (main)
       /
A ← B
       \
        D  (feature)
```

- `main`은 커밋 C, `feature`는 커밋 D를 가리킴
- 둘 다 커밋 B에서 갈라져 나옴 (공통 조상)
- **브랜치는 "이동 가능한 커밋 참조(reference)"로 개발 라인을 구분**

### 브랜치 합치기 (Merge)
```
        C ← M  (main)
       /   /
A ← B     
       \ /
        D  (feature)
```
- **병합 커밋(M)은 부모가 두 개**(C와 D)
- → 커밋 히스토리는 항상 트리가 아니라 **그래프**가 될 수 있음

---

## 20. 로컬 저장소 vs 원격(GitHub) 저장소

```
Local: A←B←C   --push-->   GitHub: A←B←C
Local: A←B←C   <--fetch--  GitHub: A←B←C
```

| 명령어 | 주요 역할 | 로컬 작업 파일에 미치는 영향 |
|---|---|---|
| `clone` | 다른 저장소로부터 로컬 저장소 생성 | 체크아웃 시 파일 생성됨 |
| `push` | 로컬 커밋을 원격으로 전송 | 업로드되어도 작업 파일 자체는 안 바뀜 |
| `fetch` | 원격 히스토리/참조를 받기만 함 | 자동으로 통합 안 됨 |
| `pull` | fetch + 현재 브랜치에 통합 | 갱신될 수 있음 |

> **편집, 커밋, 전송(push)은 서로 별개의 작업** — 커밋했다고 자동으로 GitHub에 올라가지 않음

---

## 21. 기존 저장소로 시작하기 (clone)

```bash
$ git clone REPOSITORY_URL
$ cd REPOSITORY_DIRECTORY
$ git remote -v
$ git status
```

---

## 22. 로컬 커밋을 원격에 게시 (push)

### 최초 연결 + 푸시
```bash
$ git remote add origin REPOSITORY_URL
$ git remote -v
$ git push -u origin main
```
- `origin`: 원격 저장소를 가리키는 관례적 별명
- `-u`(`--set-upstream`): 로컬 `main`과 원격 `main`을 연결

### 이후엔 짧게
```bash
$ git push
```

**전제 조건**: 원격 접근 가능 + 쓰기 권한 + 호환되는 히스토리

### 자주 겪는 문제
| 에러 | 원인 | 해결 |
|---|---|---|
| `Authentication failed` | 비밀번호 직접 입력 시도 | Personal Access Token(PAT) 사용 |
| `Permission ... denied` | 접근 권한 없는 저장소 | 협업자 초대 요청 |
| `src refspec main does not match` | 로컬 브랜치명이 `master`인데 `main`으로 push 시도 | `git branch`로 확인 후 이름 맞추기 |
| `[rejected] (fetch first)` | 원격에 이미 다른 커밋 존재 | `git pull` 먼저 실행 |
| push는 성공했는데 GitHub에 안 보임 | 커밋 자체가 없었거나(`git status`로 확인), 다른 브랜치로 올라갔거나, 새로고침 안 함 | `git log --oneline`, `git branch`, `git remote -v`로 확인 |

---

## 23. 원격 변경사항 받아오기 (pull)

```bash
$ git status
$ git pull --ff-only
$ git log --oneline
```

`--ff-only`(fast-forward only): **분기(divergence) 없이 앞으로만 진행 가능할 때만 pull 허용**, 안 되면 중단 (충돌 방지용 안전장치)

---

## 24. 실습: 커밋 결과 예측하기

```bash
# 현재 커밋 내용: A
$ echo "B" > answer.txt
$ git add answer.txt
$ echo "C" > answer.txt
$ git commit -m "Update answer"
```

### 정답

| 상태 | 내용 | 이유 |
|---|---|---|
| 새 커밋 | **B** | 스테이징된 버전이 B였음 |
| 스테이징 영역 | B | 이후 편집은 스테이징 안 됨 |
| 작업 파일 | **C** | 커밋이 작업 파일을 대체하지 않음 |
| 미스테이징 차이 | B → C | 작업 파일이 여전히 다름 |

> **핵심 교훈**: `git commit`은 "스테이징 영역"의 스냅샷만 기록하고, 이후에 파일을 또 고쳐도 그건 별개의 미스테이징 변경으로 남는다.

---

## Week 03 명령어 요약표

| 작업 | 명령어 |
|---|---|
| 현재 상태 확인 | `git status` |
| 스테이징/커밋 안 된 변경 확인 | `git diff` / `git diff --staged` |
| 스냅샷 준비 및 기록 | `git add` / `git commit` |
| 기록된 히스토리 읽기 | `git log` / `git show` |
| 커밋된 히스토리 공유 | `git push` / `git pull` |
| 스테이징/편집 되돌리기 | `git restore --staged` / `git restore` |
| 이미 추적 중인 파일 무시 | `git rm --cached` |

## Week 03 핵심 요약

> Git은 Working Tree(현재 파일) → Staging Area(다음 스냅샷 준비) → Repository(커밋 히스토리)라는 세 영역을 오가며 동작하고, `git add`는 특정 시점의 스냅샷을 "찜"하는 것이며, `git commit`은 그 찜해둔 스냅샷만 기록할 뿐 작업 파일 자체를 건드리지 않는다. 편집·커밋·push는 각각 독립된 단계이며, GitHub에 실제로 올라가려면 반드시 `git push`까지 실행해야 한다.

---

## 참고 자료
- Pro Git, Chapters 1–3: git-scm.com/book
- MIT Missing Semester 2026: missing.csail.mit.edu/2026/version-control/
- Software Carpentry: swcarpentry.github.io/git-novice/ , shell-novice
- Filesystem Hierarchy Standard 3.0: refspecs.linuxfoundation.org/FHS_3.0
- Git command reference: git-scm.com/docs
