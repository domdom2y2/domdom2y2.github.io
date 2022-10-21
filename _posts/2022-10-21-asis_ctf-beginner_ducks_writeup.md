---
title: '[ASIS CTF] Beginner ducks Writeup'
categories: [writeups, CTF]
tags: [web, CTF, writeup]
---

# [ASIS CTF] Beginner ducks Writeup

## Introduction

> **Beginner ducks (37 pts, 176 solves)**
> <br>Hiiiii, welcome to ASIS CTF. We have ducks. Check them out here Note for beginners: If you haven't played CTF before, this video might help you to understand what you have to do.
> <br>URL : http://ducks.asisctf.com:8000/
> {: .prompt-info }

![firstpage](/assets/img/beginner_ducks_writeup-2.png){: .shadow }
_문제의 첫 페이지 화면_

처음에 문제 페이지 접속하게 되면 위와 같이 랜덤한 이미지를 볼 수 있습니다.

그리고 이미지를 새 탭으로 열면 아래와 같은 주소로 접속되면서 다음과 같은 사진을 볼 수 있습니다.
```
http://ducks.asisctf.com:8000/duck?what=duckLookingAtAHacker
```

![imagepage](/assets/img/beginner_ducks_writeup-1.png){: .shadow }
_이미지 페이지_

## Code Analysis
### main.py
```python
#!/usr/bin/env python3
from flask import Flask,request,Response
import random
import re

app = Flask(__name__)
availableDucks = ['duckInABag','duckLookingAtAHacker','duckWithAFreeHugsSign']
indexTemplate = None
flag = None

@app.route('/duck')
def retDuck():
	what = request.args.get('what')
	duckInABag = './images/e146727ce27b9ed172e70d85b2da4736.jpeg'
	duckLookingAtAHacker = './images/591233537c16718427dc3c23429de172.jpeg'
	duckWithAFreeHugsSign = './images/25058ec9ffd96a8bcd4fcb28ef4ca72b.jpeg'

	if(not what or re.search(r'[^A-Za-z\.]',what)):
		return 'what?'

	with open(eval(what),'rb') as f:
		return Response(f.read(), mimetype='image/jpeg')

@app.route("/")
def index():
	return indexTemplate.replace('WHAT',random.choice(availableDucks))

with open('./index.html') as f:
	indexTemplate = f.read() 
with open('/flag.txt') as f:
	flag = f.read()

if(__name__ == '__main__'):
	app.run(port=8000)

```
{: file="main.py" }

Warm-up 문제인만큼 소스코드의 길이도 적고 간단한 것 같습니다.

what 파라미터에 들어온 값이 최종적으로 eval() 함수에 들어가는 것을 확인할 수 있습니다.

예를 들어 `http://ducks.asisctf.com:8000/duck?what=duckInABag` 에 접속하게 되면 open() 함수에는 아래와 같이 될 것입니다.
```python
with open('./images/e146727ce27b9ed172e70d85b2da4736.jpeg') as f:
	return Response(f.read(), mimetype='image/jpeg')
```

결국 eval 함수의 실행 결과는 특정한 디렉터리 경로가 되어야 한다는 것을 알 수 있습니다. 그리고 Flag의 위치는 /flag.txt 인 것을 위 문제의 소스코드를 보면 알 수 있습니다.

그리고 또 중요한 것 중에 하나는 what 파라미터를 정규식으로 정의하고 있기를 영문 대소문자와 .(dot) 밖에 허용되지 않는 다는 것입니다.

예를 들어 `/duck?what=request.args.get('params')&params=/flag.txt` 같은 형태로 하려고 해도 괄호와 따옴표를 쓸 수 없다는 것입니다.

하지만 이외에도 `Flask`의 `request`에는 다양한 속성들이 있습니다.
한번 알아보고 사용해서 Flag를 추출해보겠습니다.

## Exploit
[Flask.Request](https://flask.palletsprojects.com/en/2.2.x/api/?highlight=request#flask.Request.headers)에는 다양한 속성들이 있습니다.

다른 속성들을 사용해도 되겠지만 저는 [request.referrer](https://flask.palletsprojects.com/en/2.2.x/api/?highlight=request#flask.Request.referrer)를 이용해서 풀었습니다.

![flag](/assets/img/beginner_ducks_writeup-3.png){: .shadow }
_flag boom!_

`Referer` 헤더에 `/flag.txt` 경로를 넣어주고 `Flask`에서 `request.referrer`의 값을 `open()` 함수에 넣어줘서 Flag의 내용을 출력하게 해서 풀었습니다.