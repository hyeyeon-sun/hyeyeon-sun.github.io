---
layout: post
title: "[node.js 공부] npm 패키지 매니저"
subtitle: "패키지 매니저인 npm에 대해 알아본다."
date: 2021-08-07 10:45:13 -0400
background: '/img/bg-index.jpg'
---

🚶‍♀️Prolog
---
  개발.. 혼자하는 게 아니라고 했다. 여러사람이 만든 모듈과 라이브러리를 바탕으로 훨씬 더
 쉽게 견고한 산출물을 만들어낼 수 있다. 특히 node.js에는 **npm**이라는 강력한 패키지 관리자가
 있어 보다 쉽게 개발을 할 수 있다. 이런 npm의 개념과 활용법에 대해 차근차근 알아보기로 한다.<br>




## 1. npm 기본

 ✔ NPM :  Node Package Manager의 약자다.<br>
 ✔ node.js는 javascript를 기반으로 만들어졌으니 당연히 Date, String, Array등의 모듈을 사용할 수 있다.<br>
 ✔ npm은 node계의 앱스토어 같은 느낌. 다른 사람이 만든 module을 사용할 수 있다.
 
 
 
## 2. npm 활용

**🔎 npm init**<br>
  우리는 지금 underscore라는 다른 사람이 만든 패키지를 우리것으로 가져오려고 하는 것이다. 그런데 우리의 것도 실은 하나의 패키지이기 때문에 현재 디렉토리를 패키지로 지정하는 <code>npm init</code> 명령을 해야만 한다.

 - 이로써 우리는 우리의 디렉토리를 npm의 패키지 디렉토리로 선언을 한 것이다 !
 - 그리고 이 패키지에 대한 여러 정보를 기록도 한 것이다.

**왜필요할까 ?**  프로젝트의 의존성을 관리할 수 있으며, 나중에 프로젝트를 npm.com에 올리기 위한 초석이 될 수도 있다.
<br>

**🔎 module 설치**<br>
자 이제 모듈을 한번 설치해보자. <code>npm install underscore</code> 명령어로 underscore라는 모듈을 설치한다.

그럼 디렉토리에 변화가 생긴다 !! 

  ✔ node_modules라는 디렉토리가 생기고, 하위 디렉토리로 underscore가 생긴다. 그 안에 우리의 모듈이 위치한다.<br>
  ✔ pakage.json 파일에 dependencies(의존성)항목으로 underscore 항목이 생긴다. 
      <br>-> 이 항목이 있으면, 우리가 pakage.json이라는 파일만 있으면 언제든 underscore 파일을 포함시킬 수 있다.
    


**🔎 npm install [name] VS npm install [name]-g**
  - <code>npm install [name] -g</code> : 설치하는 패키지를 전역적으로 사용하겠다는 의미다.
  - <code>npm install [name]</code> : 설치하는 패키지를 현재 프로젝트의 부품으로 사용하겠다는 의미다.<br><br>
 

 
## 👉 모듈 정보
 실습과정에서 사용한 모듈과 명령어에 대한 정보를 정리해보았다. <br>

🔅 <code>uglifyjs [filename] -o [newfilename] -m</code> 하면 경량화된 파일을 저장할 수 있다.<br>
🔅 underscore은 보통 변수를 _로 선언하는 모듈로, javascript에서 기본적으로 제공되지 않는 다양한 기능을 제공한다.<br>
🔅  _ = require('underscore');