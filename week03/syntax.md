# 3주차 문법: Git

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 초기 설정

```
git init -b main
git config user.name "이름"
git config user.email "이메일"
```

## 기본 흐름

```
git status [--short]
git add 파일1 파일2 ...
git diff              # 워킹트리 vs 스테이징
git diff --staged     # 스테이징 vs 마지막 커밋
git commit -m "메시지"
git log --oneline [--graph]
```

## status --short 코드

```
??   추적 안 됨 (untracked)
 M   수정됐지만 스테이징 안 됨
M    스테이징됨
MM   스테이징 후 추가로 또 수정됨
```

## 되돌리기

```
git restore 파일           # 워킹트리 변경 취소 (마지막 커밋으로)
git restore --staged 파일  # 스테이징만 취소 (워킹트리는 유지)
```

## 원격 저장소

```
git remote -v
git remote add origin <URL>
git push -u origin main
git pull --ff-only
```

## 브랜치

```
git branch                 # 목록 보기
git branch 이름             # 브랜치 생성
git checkout 이름           # 브랜치 이동
git checkout -b 이름        # 생성 + 이동
git merge 브랜치이름        # 현재 브랜치에 합치기
```

## .gitignore

```
build/
reports/
*.log
```
→ 이미 추적 중인 파일엔 효과 없음. 추적 해제하려면:
```
git rm --cached 파일
```
