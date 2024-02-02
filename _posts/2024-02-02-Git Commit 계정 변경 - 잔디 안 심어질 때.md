---
title: Git Commit 계정 변경 - 잔디 안심어질 때
aliases:
  - git 계정과 git hub 계정 이 달라서 잔디가 안심어졌다...
tags:
  - git
  - github
categories:
  - etc
toc: true
toc_label: 목차
date: 2024-02-02
last_modified_at: 2024-02-02
---
회사pc 로 개인 공부나 사이드 프로젝트를 github에 올리는 경우가 있었는데 나중에 확인해 보니깐 깃허브에 잔디가 많이 비어있다 ㅋㅋ.

그래서 확인을 해보니깐 이상한 아이디로 커밋을 하고 있었다!

![](https://i.imgur.com/fAnIVyG.png)


계정이 다르다.. ㅎㅎ;

이제 터미널로 가서 지금 내 로컬 git이 뭘로 설정 되어 있나 보자.


```
git config --list
```

![](https://i.imgur.com/yZ1POhx.png)


user.name, user.email 이 전혀 다른 아이디로 되어있다. .ㅎㅎ

이 설정을 바꿔 보자 (저 창에서 안나와 지면 :wq 치고 나오면 된다. vi 에디터) 

### 1. 지금 작업중인 폴더에서만 email 변경하기


```
git config --local user.name 이름
git config --local user.email 이메일 주소
```

입력 후 확인 해 보면
![](https://i.imgur.com/sCiVWXO.png)

이렇게 2개 생겼는데 실제 커밋 할 때는 2개 중에 아래 정보로 들어가게 된다.

### 2. pc 전체에서 정보 변경하기

```
git config --global user.name 이름
git config --global user.email 이메일 주소
```

`--global` 옵션을 넣어주면 된다.

## 커밋 복구 하기

이제 잘못된 계정으로 커밋한 걸 복구 하자.


```
git log --pretty=format:"%h = %an , %ar : %s" --graph
```

![](https://i.imgur.com/7Xkqind.png)

맨 위 커밋이 계정을 바꿔줘야 하는 거다.


```
git rebase -i -r --root
```

![](https://i.imgur.com/2f6BUXg.png)
이렇게 pick 을 edit 으로 바꿔주고 :wq 로 저장해서 나와준다. 


그 후에

```
git commit --amend --author="이름<이메일>" 
```
이렇게 작성하면 작성자를 수정할 수 있다. 
vi 에디터가 나올껀데 딱히 수정할 건 없으니깐 :wq  하고 나오자.


```
git rebase --continue
git push origin main
```

![](https://i.imgur.com/oT9Eg6z.png)
![](https://i.imgur.com/Tc1gdyo.png)
이렇게 하면 끝이 난다. 

> 만약에 수정할 게 여러 개라면, edit 로 다 바꾸고 commit, rebase push 를 edit 한 만큼 반복해주면 된다.