---
title: "spring security 적용 시 모든 에러가 403인 현상"
categories:
    spring
tags:
    spring
    security
date:
    2024-05-07 21:24:50+0900
toc:
    true
toc_sticky:
    true
---

# 상황

spring security를 적용한 후, 서버에서 발생하는 모든 에러에 대해 **403** 에러가 리턴되는 현상. 

심지어 존재하지 않는 api 주소를 입력해도 **404** 에러가 아닌 **403** 에러가 발생한다. 

# 발생 이유

기본적으로 Spring boot에서는 잘못된 페이지에 대해 WhiteLabel Error Page를 표시해 준다. 

![image](https://github.com/cuzzzu1318/cuzzzu1318.github.io/assets/77597885/6cb0c918-60c5-4941-b109-f6f64e9f0625)


하지만 security 설정 때문에 error 페이지에도 authentication을 요구하게 되었다!

![image](https://github.com/cuzzzu1318/cuzzzu1318.github.io/assets/77597885/a304ee75-206c-4c32-8429-95f7e047345e)

# 해결

해결 방법은 간단하다. error페이지에 대한 요청을 허용해 주면 에러에 대한 응답이 정상적으로 반환되게 된다

![image](https://github.com/cuzzzu1318/cuzzzu1318.github.io/assets/77597885/0a824f86-8715-4cf3-bdd4-6df5cf41831c)
