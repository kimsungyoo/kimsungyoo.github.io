---
category: blog
layout: post
author: kimsungyoo
title: "Python을 이용한 한글 이름을 영어로 변환하기"
description: "한글 이름의 영문 로마자 변환하기를 python으로 구현합니다."
date: 2019-08-28 23:12
headerImage: false
tag:
- python
- english
- translate
---
한글 이름을 영문 로마자로 변경하는 툴이 있을까 싶어서 찾아보니 역시 있었습니다.
(세상엔 좋은 사람들이 참 많습니다.)<br />

- [네이버 한글 이름 로마자 표기](https://dict.naver.com/name-to-roman/translation/?where=name){:target="_blank"}

이름을 영어로 변환하는 것은 문장을 영어로 번역하는 것과는 다른 영역이라 일반적인 번역기로는 목표를 달성하기 힘듭니다.
네이버 한글 이름 로마자 변환 도구는 한글 이름에 대한 여러 영문 표기법들을 사용 빈도와 함께 제공해 주는 훌륭한 툴입니다.
일반적으로 사용하기에는요...
<br />
<br />
하지만 이 사이트를 개발에 그대로 사용하기에는 무리가 있습니다.
selenium을 이용해서 브라우저를 통한 크롤링을 할 수도 있지만 일단 너무 무겁고, 서버 환경에서는 사용하지 못할 수도 있습니다.
하지만 인터넷의 예제 코드는 selenium을 사용한 것 밖에 없네요. (...)
<br />
<br />
그래서 직접 만들어 보았습니다. 개발자니까요.

----

## 네이버 API를 이용한 구현

- 네이버 한글인명 - 로마자 변환 API [[링크](https://developers.naver.com/products/roman/){:target="_blank"}]

네이버가 또 좋은 일을 했습니다. (만세!)
API를 이용하려면 사용신청을 해서 CLIENT ID와 SECRET KEY를 발급받아야 합니다.
페이지의 아랫부분에 있는 "이용 신청" 버튼을 통해서 이용신청을 합니다.
개발 가이드를 보니 python 예제도 있네요.
예제를 살짝 고쳐서 실행해 봅니다.

사용한 파이썬 버전은 3.6입니다.

```python
import urllib.request

client_id = "<네이버에서 발급받은 CLIENT ID>"
client_secret = "<네이버에서 발급받은 CLIENT SECRET>"

encText = urllib.parse.quote("배수지")
url = "https://openapi.naver.com/v1/krdict/romanization?query=" + encText

request = urllib.request.Request(url)
request.add_header("X-Naver-Client-Id",client_id)
request.add_header("X-Naver-Client-Secret",client_secret)
response = urllib.request.urlopen(request)
rescode = response.getcode()

if(rescode==200):
    response_body = response.read()
    print(response_body.decode('utf-8'))
else:
    print("Error Code:" + rescode)
```

예제는 홍길동으로 했지만 저는 배수지로 검색해 보겠습니다.
 
```
{"aResult":[{"sFirstName":"\ubc30","aItems":[{"name":"Bae Sooji","score":"99"},{"name":"Bae Suji","score":"78"},{"name":"Bae Soojee","score":"27"},{"name":"Bae Sujee","score":"21"}]}]}
```

뭐 결과물은 잘 나온 것 같은데 영문 이름만 뽑아 보고 싶습니다.<br />
코드를 살짝 고쳐봅니다.
 
```python
import json
import urllib.request

client_id = "<네이버에서 발급받은 CLIENT ID>"
client_secret = "<네이버에서 발급받은 CLIENT SECRET>"

encText = urllib.parse.quote("배수지")
url = "https://openapi.naver.com/v1/krdict/romanization?query=" + encText

request = urllib.request.Request(url)
request.add_header("X-Naver-Client-Id",client_id)
request.add_header("X-Naver-Client-Secret",client_secret)
response = urllib.request.urlopen(request)
rescode = response.getcode()

if(rescode==200):
    response_body = response.read()
    json_dict = json.loads(response_body.decode('utf-8'))
    result = json_dict['aResult'][0]
    name_items = result['aItems']
    names = [name_item['name'] for name_item in name_items]
    print(names)
else:
    print("Error Code:" + rescode)
```

결과물

```
['Bae Sooji', 'Bae Suji', 'Bae Soojee', 'Bae Sujee']
```

훌륭하네요. 네이버가 일을 참 잘 하는 것 같습니다.
네이버 API 페이지를 살펴보니 1일 25,000글자만 변경이 가능하다고 합니다.
한글 이름은 보통 3글자니까 하루에 8333명 정도 변환이 가능하네요.
이 이상 사용하고 싶으면 네이버 클라우드를 이용해서 이용한도 상향(신용카드인가!) 신청을 해야 합니다.

귀찮습니다.

---

## 웹 크롤러 기반의 구현

포스팅의 처음에 소개한 사이트를 이용해서 크롤링을 해봅니다.
다행히 이 사이트는 검색에 GET방식을 사용하고 있어서 주소창에 값만 바꿔주면 원하는 검색 페이지를 가져올 수 있습니다.

```
https://dict.naver.com/name-to-roman/translation/?query=배수지
```

이런 식입니다. 이름 값만 바꾸면 다른 이름으로도 검색이 가능합니다.<br />
크롤링 데이터를 쉽게 가져오기 위해서 BeautifulSoap 라이브러리를 사용하겠습니다.
혹 설치가 안되어 있다면

```shell
$ pip install bs4
```

selenium은 무거우니까 urllib을 이용해서 구현해 보겠습니다.

```python
import urllib
from urllib import parse
from urllib.request import Request, urlopen

from bs4 import BeautifulSoup

naver_url = 'https://dict.naver.com/name-to-roman/translation/?query='
name_url = naver_url + urllib.parse.quote("배수지")

req = Request(name_url)
res = urlopen(req)

html = res.read().decode('utf-8')
bs = BeautifulSoup(html, 'html.parser')
name_tags = bs.select('#container > div > table > tbody > tr > td > a')
names = [name_tag.text for name_tag in name_tags]

print(names)
```

결과물

```
['Bae Sooji', 'Bae Suji', 'Bae Soojee', 'Bae Sujee']
```

API를 이용한 결과물과 동일하게 나왔습니다.

웹크롤링을 이용하면 API 키 발급도 안해도 되고 25,000글자 이상도 변환할 수 있습니다.
하지만 API는 이름 관련 데이터만 가져오는 반면,
웹크롤링은 웹페이지를 보여주기 위한 필요 없는 데이터까지 (게다가 양도 몇배나 많음!) 가져옵니다.
가능하다면 API 사용신청을 해서 한도 상향 신청을 해서 사용하는 것이 좋은 방법이겠지요.

관련 소스코드는 GitHub에 올려두었습니다.
[https://github.com/kimsungyoo/python-korean-to-roman](https://github.com/kimsungyoo/python-korean-to-roman){:target="_blank"}

---

## 결론

세상엔 좋은 일을 해둔 사람이 많고,<br />
오픈소스는 쓸려고 하면 꼭 하나가 아쉽습니다. ㅠㅜ<br />
하지만 괜찮습니다. 직접 개발하면 되니까요.

배수지 만세!
