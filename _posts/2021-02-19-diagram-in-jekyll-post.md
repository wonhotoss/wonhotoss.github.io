---
layout: richpost
title: "jekyll 포스트에 다이어그램 삽입하기"
---

{{ page.title }}
개발 관련 글을 쓸 때는 다이어그램이 필요할 때가 많다. 업무관계로 구글 문서로 글을 쓸 때는 구글 드로잉을 중간에 넣는 식이었는데, github page를 만들면서 다이어그램을 첨부하려고 찾은 방법들을 정리해본다.
<br/><br/>
# 이미지 첨부
그냥 이미지를 첨부한다. 어딘가에서 그려와서 싸이트에 적재하고 문서 내에서 참조한다. 참조하는 방법은 여러가지 있지만 제일 간단해 보이는 markdown 문법을 소개한다.

![펭귄](/assets/images/tux.png)

    ![펭귄](/assets/images/tux.png)

요새도 visio 쓸 일은 없을 것 같고, 구글 드로잉이나 어딘가 편한 곳에서 그려오면 된다지만 작업을 두 군데에서 해야 하고, 뭐 하나 수정하면 익스포트해서 가져오고 하는지라 불편하다. 차라리 [ascii art](http://stable.ascii-flow.appspot.com/) 같은 게 나을지도.
<br/><br/>
# 구글 드로잉 
구글 드로잉을 만들고 웹에 게시한 후 문서에 적재. 이전의 포스트에서 이 방법을 사용함.
<img src="https://docs.google.com/drawings/d/e/2PACX-1vSKjCC2-CN1iw8gUkMSxmxfM_MkUZLwSCmXYFRal8yMH03wYggNrgw6L_bGB9f_gq7DpwejjG-6O1y6/pub?w=480&amp;h=360">

    <img src="https://docs.google.com/drawings/d/e/2PACX-1vSKjCC2-CN1iw8gUkMSxmxfM_MkUZLwSCmXYFRal8yMH03wYggNrgw6L_bGB9f_gq7DpwejjG-6O1y6/pub?w=480&amp;h=360">
나쁘지 않다. 장점이라면...
- 구글 드로잉이 워낙에 그리기 좋음
- 블로그 배포 없이 그림만 수정가능
- 만든 그림은 다른 포스트나 구글 문서에서도 참조가능


그래도...
- 기능이 풍부한 반면 간단한 다이어그램에는 무겁고 번거롭다.
- 어쨋든 두 군데에서 편집해야 함.

<br/><br/>
# Mermaid
markdown이 html을 그냥 포함해도 되는 걸 모르고 있었다 바보였다...자바스크립트로 SVG를 생성하는 다이어그램이나 차트 라이브러리를 써보면되지 않을까? kramdown이 자바스크립트가 포함된 포스트를 어떻게 컴파일할까? 마크다운 뷰어가 자바스크립트를 어디까지 돌려줄까? 결과는...

<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true});</script>

<div class="mermaid">
    graph TD
    A[Client] --> B[Load Balancer]
    B --> C[Server1]
    B --> D[Server2]
    B --> E[Server3]
</div>

    <script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
    <script>mermaid.initialize({startOnLoad:true});</script>

    <div class="mermaid">
        graph TD
        A[Client] --> B[Load Balancer]
        B --> C[Server1]
        B --> D[Server2]
        B --> E[Server3]
    </div>

걍 넣으니깐 된다;;;

![펭귄](/assets/images/스크린샷20210220.png)

심지어 vscode에서 live preview까지 넘므 좋다...처음 vscode에서 편집하면 security warning이 뜰 텐데, 일단 disable해 주자. 아래 스크린샷 참조. 

![insecure](/assets/images/스크린샷insecure.png)
![disabled](/assets/images/스크린샷disable.png)


장점은..
- 간단한 다이어그램은 간단하게. 간단한 선언적 문법으로 다이어그램 생성.
- 편집기 옮겨다닐 필요없이 마크다운 에디터만으로 충분. 글쓰기의 연장선에서 그림을 그리고 고칠 수 있다는 점이 좋다.

단점은...
- 다이어그램을 이쁘게 그리는 데에는 한계가 있는 듯.

아래의 stub 코드를 포스트마다 반복해야 한다는 게 문제인데...

    <script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
    <script>mermaid.initialize({startOnLoad:true});</script>

Liquid로 어떻게 되겠지만 마크다운 뷰어가 liquid를 알리가 없으니 프리뷰가 안되서 아쉽다.
<img src='../assets/images/tux.png'>
