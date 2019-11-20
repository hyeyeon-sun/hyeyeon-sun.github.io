---
title: "setting up a github pages with jekyll"
date: 2010-11-20 08:26:28 -0400
categories: jekyll update
---

1. username.github.io 라는 repository를 만든다.
2. jekyll에서 제공하는 테마를 선택한다.
 - jeky검색 -> topics category -> jekyll themes
3. username.github.io 에 원하는 테마의 repository에 있는 _congif.yml을 복사해 온다.
 - 이때 remote_theme : 저자/테마 이름 (한 줄 추가)
 - url : "username.github.io"(변경)
 - base url : ""(변경)
 4. index.html파일도 복사해 온다. 이건 추가나 변경 안해도 된다.
 (테마에 따라 posts.md파일도 복사해야 할 수도 있다)
 (제대로 동작하지 않으면 설명을 읽어보며 다른 파일도 복사해온다)
 
**완성**
 
 포스트 작성하기
 
 repository에서
_posts/YYYY-MM-DD-name-of-post.md 제목으로 새로운 파일을 만든다.

파일 맨 처음에는 다음과 같이 글의 제목과 날짜 등을 적어준다.

*/
---
title: "Welcome to Jekyll!"
date: 2019-11-20 08:26:28 -0400
categories: jekyll update
---

밑에는 자유롭게 원하는 내용을 적는다 !

**포스트 작성 끝**

username.github.io에 접속해 확인한다!
