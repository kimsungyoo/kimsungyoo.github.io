---
category: blog
layout: post
author: kimsungyoo
title: "Django가 Password를 생성하는 방법"
description: "Django에서 settings.py에 정의된 SECRET_KEY는 어디에 쓰는지, 패스워드는 어떤 방법을 통해 생성되는지, 그리고 SECRET_KEY와 Password 사이의 상관관계는 무엇인지 알아봅니다."
date: 2019-09-12 18:35
headerImage: false
tag:
- dev
- Django
- python
- security
- password
---
Django 프로젝트를 생성하면 settings.py에는

```python
# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = '<your secret key>'
```

```SECRET_KEY```  와 이와 관련된 보안 경고 주석이 있습니다.

staging 환경을 새로 구성하면서 보안 조치를 위해서 production 환경과 다른 ```SECRET_KEY```  값을 넣어주었고, 그에 따라 기존 DB를 복사해서 넣었을 때 예전 패스워드를 사용하지 못할 것으로 예상했습니다.

하지만 그렇지 않았습니다. 기존 패스워드를 입력해도 로그인이 "잘" 되었습니다.

그렇다면 ```SECRET_KEY``` 는 무엇이며, Django 패스워드는 어떠한 방식으로 생성되는지, 그리고 ```SECRET_KEY``` 와 Django 패스워드 사이의 상관관계는 없는 것인지에 대한 의문이 생겼고, 여러 문서와 코드를 통해 발견한 바를 공유합니다.

## Django는 SECRET_KEY를 어디에 사용하는가?

------

- 이에 대해 잘 정리된 블로그 글 : [[링크](https://wayhome25.github.io/django/2017/07/11/django-settings-secret-key/){:target="_blank"}]

[공식문서](https://docs.djangoproject.com/en/1.11/ref/settings/#std:setting-SECRET_KEY){:target="_blank"} 의 설명에 따르면 ```SECRET_KEY``` 는

- session
- cookie
- PasswordResetView
- [암호화된 서명](https://docs.djangoproject.com/en/1.11/topics/signing/){:target="_blank"}

에서 사용한다고 합니다. 그런데 눈을 씻고 찾아봐도 Password를 생성하는데 사용한다고 적혀 있지 않습니다. PasswordResetView에서 token을 발행하는데 사용한다고 나와 있긴 하지만 token이야 다시 발행하면 그만이고, ```SECRET_KEY``` 가 변하면 session이나 cookie가 끊어지긴 하겠지만 어차피 staging 환경을 다시 구성하려고 하는 것이므로 신경을 쓸 필요가 없을 것 같습니다. 암호화된 서명 관련해서도 위와 마찬가지입니다.

그렇다면 ```SECRET_KEY``` 의 값을 바꾸어도 기존 패스워드를 계속 사용할 수 있는가? 이에 대한 확신을 얻기 위해서는 Django가 패스워드를 어떻게 생성하는지 알아 볼 필요가 있을 것 같습니다.

## Django가 패스워드를 생성하는 방법

------

- 공식 문서 : [Password management in Django](https://docs.djangoproject.com/en/1.11/topics/auth/passwords/){:target="_blank"}

문서에 따르면 장고의 패스워드는 기본적으로 PBKDF2 알고리즘을 사용하고 ([NIST](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-132.pdf){:target="_blank"} 의 권장사항이라고 합니다.)

```
<algorithm>$<iterations>$<salt>$<hash>
```

위의 형태와 같이 저장된다고 합니다.

User를 생성해서 DB의 ``` auth_user ``` 테이블에 저장된 ``` password ``` 를 보면

```
pbkdf2_sha256$36000$<my salt>$<my hash>
```

라고 되어있네요. pbkdf2_sha256 알고리즘을 사용했고 36000번 iteration 해서 패스워드를 생성했습니다. 별 다른 설정 없이도 NIST의 권장사항을 잘 준수했습니다. (Django 만세!)

공식문서를 보면

```python
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
    'django.contrib.auth.hashers.BCryptPasswordHasher',
]
```

알고리즘을 생성하는 ``` Hasher ``` 는 이렇게 기본 설정이 되어 있고, 설정 값의 처음에 적혀 있는 것을 사용한다고 합니다. 여기서는 ``` PBKDF2PasswordHasher ``` 를 사용하게 되는 것이겠죠. 위의 값은 1.11 문서인데 2.2의 문서에서는 ``` BCryptPasswordHasher ``` 가 빠져있는 점도 흥미롭습니다. ``` Argon2PasswordHasher ``` 는 2015 패스워드 해싱 대회 우승한 알고리즘이라고 하니 다음에 프로젝트를 생성하게 되면 이걸 사용하는 것도 한번 고려해 봐야겠습니다.

그래서 ``` PBKDF2PasswordHasher ``` 의 코드를 보면

```python
class PBKDF2PasswordHasher(BasePasswordHasher):
    """
    Secure password hashing using the PBKDF2 algorithm (recommended)

    Configured to use PBKDF2 + HMAC + SHA256.
    The result is a 64 byte binary string.  Iterations may be changed
    safely but you must rename the algorithm if you change SHA256.
    """
    algorithm = "pbkdf2_sha256"
    iterations = 36000
    digest = hashlib.sha256

    def encode(self, password, salt, iterations=None):
        assert password is not None
        assert salt and '$' not in salt
        if not iterations:
            iterations = self.iterations
        hash = pbkdf2(password, salt, iterations, digest=self.digest)
        hash = base64.b64encode(hash).decode('ascii').strip()
        return "%s$%d$%s$%s" % (self.algorithm, iterations, salt, hash)
```

``` iterations ```  의 기본값은 36000 번이고, ``` algorithm ``` 은 pbkdf2_sha256 이네요.

``` hash ``` 는 password, salt, iterations, digest를 이용해서 생성합니다.

그리고  ``` salt ``` 는 ``` PBKDF2PasswordHasher ``` 에 정의되어 있지 않아서 부모 클래스인 ```BasePasswordHasher ``` 를 보면

```python
class BasePasswordHasher(object):
    """
    Abstract base class for password hashers

    When creating your own hasher, you need to override algorithm,
    verify(), encode() and safe_summary().

    PasswordHasher objects are immutable.
    """
    
    ...

    def salt(self):
        """
        Generates a cryptographically secure nonce salt in ASCII
        """
        return get_random_string()

```

random string 임을 알 수 있습니다.

따라서 ``` SECRET_KEY ``` 와 관련있는 부분은 전혀 없음을 알 수 있습니다.

## 결론

------

 ``` SECRET_KEY ``` 의 값을 바꾸어도 기존의 패스워드를 그대로 사용할 수 있습니다. 

``` PASSWORD_HASHERS ``` 를 재정의하지 않는다면 Django의 버전을 바꾸어도, 프로젝트 명을 바꾸어도, repository를 바꾸어도 기존의 패스워드를 그대로 사용할 수 있습니다.

즐코딩하세요.
