---
title: "[pwnable.kr] collision"
categories:
- wargames
- pwnable.kr
tags:
- wargames
- pwnable.kr
---

[pwnable.kr 문제 풀이 목록 바로가기](/categories/pwnable-kr)


---
## 문제 설명

![문제 설명]({{ 'assets/post_img/collision_question.png' | relative_url }})

> Daddy told me about cool MD5 hash collision today.
> 
> I wanna do something like that too!
> 
> 아빠가 오늘 MD5 해시 충돌에 대한 멋진 얘기를 해주셨어요.
> 
> 나도 그런거 하고싶어요!
> 


문제를 풀기 전에 주어진 접속 정보(`ssh col@pwnable.kr -p 2222 (pw:guest)`)로 접속 했을 때 아래와 같은 파일 목록을 확인할 수 있습니다.
```bash
col@pwnable:~$ ls -l
total 16
-r-sr-x--- 1 col_pwn col     7341 Jun 11  2014 col
-rw-r--r-- 1 root    root     555 Jun 12  2014 col.c
-r--r----- 1 col_pwn col_pwn   52 Jun 11  2014 flag
```

이번 문제도 주어진 파일 중 `col.c` 파일을 읽어보고 해석하여 적절한 페이로드를 삽입하여 해결을 시도할 수 있습니다.

---

## 문제 풀이
### 코드 분석
`col.c`를 열어보면 다음과 같은 소스 코드를 확인할 수 있습니다.
중요한 코드드를 해석해 보면서 문제에 접근을 해보겠습니다.

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
    int* ip = (int*)p;
    int i;
    int res=0;
    for(i=0; i<5; i++){
        res += ip[i];
    }
    return res;
}

int main(int argc, char* argv[]){
    if(argc<2){
        printf("usage : %s [passcode]\\n", argv[0]);
        return 0;
    }
    if(strlen(argv[1]) != 20){
        printf("passcode length should be 20 bytes\\n");
        return 0;
    }

    if(hashcode == check_password( argv[1] )){
        system("/bin/cat flag");
        return 0;
    }
    else
        printf("wrong passcode.\\n");
    return 0;
}
```

#### main()
우선 메인 함수에서 어떤 동작을 하며, 어떤 행위를 했을 때 플래그를 획득할 수 있는지 확인해 볼 필요가 있습니다.

**15 행** : `if(argc<2)`

- 프로그램을 실행 시킬 때 넘겨주는 argument의 개수가 2개 미만일 경우 프로그램을 종료 시킵니다.
	
**19 행 ~ 22행** : 

```c
    if(strlen(argv[1]) != 20){
        printf("passcode length should be 20 bytes\\n");
        return 0;
    }
```

- `argv[1]` 사용자의 입력으로 들어온 argument가 20과 같지 않다면 프로그램을 종료 시키는 코드가 됩니다.

**24 행 ~ 27 행** :
```c
    if(hashcode == check_password( argv[1] )){
        system("/bin/cat flag");
        return 0;
    }
```

- `hashcode` 와 `check_password( argv[1] )`의 실행 결과가 같다면  `flag`파일을 출력합니다.
- `hashcode`는 상단에 선언되어 있으며, `0x21DD09EC`값을 가지고 있습니다.

main의 분석 결과로 봤을 때 `check_password()`에 어떤 값을 인자로 주어야 할지 분석할 필요가 있습니다.

#### check_password()
`check_pasword` 함수를 분석하고 어떤 값을 return 하는지 확인해 봐야합니다.

**5 행** : `int* ip = (int*)p;`

- parameter(사용자가 입력한 값)을 int형 포인트 변수로 변환하여 `ip`변수에 저장합니다.
	- 기존 char형 포인트 변수는 1byte의 크기를 가지고 있기 때문에 그대로 사용하게 되면 `for문`에서 1byte씩 읽게 됩니다.
	- 하지만 형변환을 실시한 int 형 포인트 변수는 4byte의 크기를 가지고 있어 `for문`안에서 4바이트 씩 읽어 진행하게 됩니다.

**8 행 ~ 10 행** :
```c
    for(i=0; i<5; i++){
        res += ip[i];
    }
```

- `i=0`으로 총 5번 반복을 실행합니다.
- `ip[i]`를 통해 입력 받은 문자열을 4byte씩 읽어 res에 저장하게 됩니다.

### 결론
1. 입력 받은 20byte 문자열을 4byte씩 정수 형태로 읽어들여 res에 더하는 과정을 5번 반복합니다.
2. res의 결과가 `0x21DD09EC`가 되어야 합니다.
3.  res 값이 `0x21DD09EC`( 10진수, 568134124)가 되기 위해서 다음과 같은 식의 결과로 문자열을 입력할 수 있습니다.
	- `0x21DD09EC`/ 5 = `0x6C5CEC8`(113626824)의 나머지 4의 값이 됩니다.
4. 3번의 연산결과를 토대로 `0x6C5CEC8` 문자열이 4번, `0x6C5CEC8 + 4`를 1번 입력 했을 때, 연산 결과는 `0x21DD09EC`가 나오게 됩니다.
	- `0x6C5CEC8 * 4 + 0x6C5CEC8 + 4` = `0x21DD09EC`
	- `0x6C5CEC8 * 4 + 0x6C5CECC` = `0x21DD09EC`
5. 즉, `0x6C5CEC8 * 4 + 0x6C5CECC`의 페이로드를 넘겼을 때 플래그를 획득할 수 있습니다.


## 결과
결론을 토대로 `0x6C5CEC8 * 4 + 0x6C5CECC`값을 페이로드로 넘겨야 합니다.

코드를 작성해서 넘기는 방법이 존재하지만, 간단한 페이로드의 경우 다음과 같이 한 줄 스크립트로 넘겨 줄 수 있습니다.

> 주의해야할 점은 문자열을 넘길땐 리틀 엔디안 방식으로 넘겨야 한다는 점을 기억해야 합니다.
> 

```bash
col@pwnable:~$ ./col `python -c 'print "\xC8\xCE\xC5\x06"*4 + "\xCC\xCE\xC5\x06"'`
daddy! I just managed to create a hash collision :)
```
