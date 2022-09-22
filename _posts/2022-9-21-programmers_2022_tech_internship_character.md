---
layout: post
title: "[Programmers] 2022 KAKAO TECH INTERNSHIP - 성격유형 검사하기(python)"
subtitle: "프로그래머스 코테 연습"
date: 2022-09-20 10:45:13 -0400
background: '/img/bg-index.jpg'
---

• Prolog
---
 (바쁘신 분들은 pass 하세유)<br>
  프로그래머스 양궁 문제로 오랫동안 씨름하다가 너무 진도가 안나가서 일단 level1문제를 간만에 풀어보았다. 앞으로 매일 한문제 이상은 풀고 포스팅을 올리는게 목표이므로 습관을 들일때 까지는 당분간 level1,2문제를 풀고, 점차 난이도를 높여가는 식으로 해야겠다.

• Problem
---
[![성격유형 검사하기](/assets/image/photo.png){: width="100%" }](https://school.programmers.co.kr/learn/courses/30/lessons/118666)
↑클릭 시 문제로 이동
<br>
mbti와 같이 설문지를 주면 그 결과로 성격유형을 반환하는 문제다.

• Solution
---
설문지를 기반으로 점수를 매기는 부분과 매겨진 점수로 성격유형을 나타내는 부분으로 나누었다.

```python
def solution(survey, choices):
    character = {'R': 0 , 'T' : 0 , 'C': 0 , 'F' : 0, 'J': 0 , 'M' : 0, 'A': 0 , 'N' : 0}
    score = [3,2,1,0,1,2,3]
    for i in range(len(survey)):
        if choices[i] > 4:
            character[survey[i][1]] += score[choices[i]-1]
        elif choices[i] < 4 :
            character[survey[i][0]] += score[choices[i]-1]
```
character라는 딕셔너리를 생성하고, 4보다 클경우 우측의 성격유형에 값을 더하고, 4보다 작을 경우 좌측의 성격유형에 값을 더한다. 값을 더할때는 score라는 1차원 배열로 쉽게 계산할 수 있도록 했다.

```python
    i = 0
    answer = ""
    for c in character:
        if i%2 == 1:
            if character[c] == character[pc]:
                answer += min(c, pc)
            else:
                add = c if character[c] > character[pc] else pc
                answer += add
        i += 1
        pc = c
        
    return answer
```
다음으로는 딕셔너리의 짝수값마다 순환하며 이전의 유형과 비교를 해 성격유형을 차례대로 판단한다. 이때 딕셔너리를 인덱스로 호출하는 법을 몰라 위처럼 i라는 변수를 두어 카운트 했는데, 찾아보니 **enumerate**라는 함수로 값을 하나하나 꺼낼 수 있다고 한다. 다음에 시도해보아야겠다.

• result
---
![성격유형 검사하기](/assets/image/sucess.png){: width="50%" }

• Source Code
---
```python
def solution(survey, choices):
    character = {'R': 0 , 'T' : 0 , 'C': 0 , 'F' : 0, 'J': 0 , 'M' : 0, 'A': 0 , 'N' : 0}
    score = [3,2,1,0,1,2,3]
    
    for i in range(len(survey)):
        if choices[i] > 4:
            character[survey[i][1]] += score[choices[i]-1]
        elif choices[i] < 4 :
            character[survey[i][0]] += score[choices[i]-1]
    
    i = 0
    answer = ""
    for c in character:
        if i%2 == 1:
            if character[c] == character[pc]:
                answer += min(c, pc)
            else:
                add = c if character[c] > character[pc] else pc
                answer += add
        i += 1
        pc = c
        
    return answer
```