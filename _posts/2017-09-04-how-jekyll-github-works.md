---
layout: post
title: "jekyll과 github과 github page"
tags: 
    - jekyll 
    - github page
---

<img src="https://docs.google.com/drawings/d/e/2PACX-1vRKioDaDWK5z53ARwSD0YG0MZ8IfD6M8TXGgBN1VLfFfPhTWX1DgaG8X-1OWsabMtCNkumDFiQdgjL8/pub?w=1440&amp;h=1080">

- jekyll은 구조화된 일련의 데이터를 정적 싸이트로 빌드하는 툴. 정적 싸이트라고 하지만 뭐든 될 것 같진 않고 일단 블로그 지향적. 포스트, 컨피그, 테마 등등을 편집한 후 빌드하면 각각의 포스트 페이지들과 인덱스 페이지 문서를 맹글어준다. 이 문서들을(_sites) 어디엔가 올려서 호스팅하면 블로그 완성.

- github은 jekyll 엔진을 내장하고 있고, 특정 이름의 저장소(username.github.io)를 github page로 부르며 특별히 관리한다. 이 저장소의 master브랜치를 커밋하면, 이 저장소를 jekyll데이터로 간주하고 빌드한 후 결과를 https://username.github.io 에 게시한다.

- github 블로그를 만들려면 굳이 로컬에 jekyll을 설치하고 푸쉬하고 해도 되지만, 그냥 적당한 곳에서 fork해오거나 아예 손으로;;; jekyll 데이터를 만들어도 된다. 만든 이후엔 굳에 로컬에서 편집할 필요 없이 github의 웹 편집기로 포스트만 커밋하면 github이 빌드해서 게시함.
