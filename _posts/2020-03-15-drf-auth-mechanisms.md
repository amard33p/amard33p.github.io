---
title: "Authentication mechanisms in Django REST Framework"
layout: post
excerpt: "Let's have a look at the authentication mechanisms available in Django REST Framework "
tags:
- django
- rest-framework
- jwt
---

## Requirements

* `pip3 install djangorestframework djangorestframework-simplejwt`
* Clone appropriate release branch of <https://github.com/amard33p/drfauth101/>


### Basic Authentication
* Most basic form. Must be implemented over HTTPS.

Usage  
```
git clone -b 0.2 https://github.com/amard33p/drfauth101
# Create superuser, make migrations and runserver
curl -X GET -u root:toor http://127.0.0.1:8000/home/
```


### Session Auth
* Client authenticates with its credentials and receives a `session_id`.
* `sessionid` can be stored in a cookie and this is attached to every subsequent outgoing request.
* Server stores `sessionid` in memory or DB and validates it on every request. Not stateless.
* Additional checks required to ensure `sessionid` leaks can be nullified.
* Every request extends expiry of `sessionid`.
* Not scalable in a load balanced environment.
* High memory usage or DB hits.

Usage  
1. `git clone -b 0.3 https://github.com/amard33p/drfauth101`
2. Login to the admin site in Chrome
3. Under DevTools > Application > Cookies get `sessionid` cookie value
4. `curl -X GET -b 'sessionid=<COOKIE_VALUE>' http://127.0.0.1:8000/home/`


### Token Auth
* No session persisted on server side.
* One token for all sessions and tokens have no expiry by default.
* All requests result in database access to fetch the user associated with the token.

Usage  
```
git clone -b 0.4 https://github.com/amard33p/drfauth101
# Generate the token
curl -X POST http://127.0.0.1:8000/api-token-auth/ -d "username=root&password=toor"
# Pass the token as header
curl -X GET http://127.0.0.1:8000/home/ -H 'Authorization: Token 731ce64bb173d70d90f0748a60aae839c12c37e4'
```


### JWT Auth
* The JWT is acquired by exchanging an username + password for an access token and an refresh token.
* The access token is usually short-lived (~5mins).
* The refresh token lives a little bit longer (~24hrs). Can be used to get new access token after expiry.
* Tokens stored client-side. Server decodes token to get expiry and only then checks user associated with token in DB.
* Fields in JWT:
  - Header - Type and Signing Algorithm
  - Payload - Token Type (Access/Refresh), Expiry, UserID
  - Signature - Using above algorithm on header base64 + payload base64 + SECRET_KEY

Usage  
```
git clone -b 0.5 https://github.com/amard33p/drfauth101
# Generate the tokens
curl -X POST http://127.0.0.1:8000/api/token/ -d "username=root&password=toor"
# Pass access token as header
curl -X GET http://127.0.0.1:8000/home/ -H 'Authorization: Bearer <ACCESS_TOKEN>'
```


_References:_  
- <https://stackoverflow.com/questions/6068113/do-sessions-really-violate-restfulness?rq=1>
- <https://dev.to/thecodearcher/what-really-is-the-difference-between-session-and-token-based-authentication-2o39>
- <https://security.stackexchange.com/questions/81756/session-authentication-vs-token-authentication>
- <https://simpleisbetterthancomplex.com/tutorial/2018/11/22/how-to-implement-token-authentication-using-django-rest-framework.html>
- <https://stackoverflow.com/questions/31600497/django-drf-token-based-authentication-vs-json-web-token>
- <https://auth0.com/learn/token-based-authentication-made-easy>
- <https://auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them>
- <https://simpleisbetterthancomplex.com/tutorial/2018/12/19/how-to-use-jwt-authentication-with-django-rest-framework.html>
