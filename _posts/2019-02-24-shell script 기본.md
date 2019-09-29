---
layout: post
title: "shell script 기본"
categories:
  - Posts
tags:
  - shellscript
last_modified_at: 2019-02-241T12:57:42+09:00
---

bash shell을 사용해 스크립트 작성 및 실행을 해보자.



**기본 스크립트 (basic1.sh)**

```shell
#!/bin/sh
echo "사용자 이름 :" $USERNAME
echo "호스트 이름 :" $HOSTNAME
```



스크립트 실행

```
sh basic1.sh
```



또는 스크립트 파일을 실행 가능한 속성으로 변경 후 실행할 수 있음
```
chmod 777 basic1.sh
./basic1.sh
```



### 변수
모든 변수에 넣는 값은 모두 `문자열` 로 취급한다.

`대소문자 구분`, 변수 대입 (=)에 공백이 없어야 함.



$를 붙임으로써 변수에 있는 값을 출력할 수 있다.

```
var="test"
echo $var
```

> 출력값: test



**주의사항**

`var = "test"` 와 같이 공백을 넣어주면 var, =, "test"가 각 인수로 되기 때문에 공백을 꼭 제거해주어야 한다.



$앞에 \를 붙이거나 ''를 통해 변수 값이 아닌 문자를 그대로 출력한다.

```
echo \$var
echo '$var'
```

> 출력값: $var, $var



read 명령을 통해 해당 변수에 값을 콘솔로 입력받을 수 있다.

```
read var
echo $var
```

> 출력값: 키보드에 입력한 값이 그대로 출력됨



### 숫자 계산

모든 변수의 값은 문자열로 취급하기 때문에, 숫자 연산 (+, -, *, /) 를 하기 위해 `expr` 키워드를 사용한다.*

추가로 곱하기 (*)랑 괄호는 앞에 \ 값을 붙여줘야 한다. (escape)

참고로 각 단어들은 모두 공백이 있어야 한다.

```
num1 = 100
num2 = 'expr $num1 + 200'
echo num2

num3 = 'expr \( $num1 \* 200 \) / 10'
echo num3

num4 = $((1 + $num3))
echo num4
```



### 파라미터 변수

파라미터 변수는 $0, $1, $2 등의 형태를 가진다.
예를 들어 `yum -y install gftp`를 실행한다고 했을 때, 각 아래와 같이 매칭 된다.

```
yum --> $0

-y --> $1

install --> $2

gftp --> $3

yum -y install gftp --> $* (전체 파라미터를 뜻함)
```



#### 그 외에도

$#은 파라미터의 수를 뜻한다.

$@는 파라미터의 배열을 뜻한다. 

$?는 명령의 종료 상태 값을 나타낸다. 정상종료가 되었다면 0, 아니면 다른 값을 반환한다.



**$ 문자 정리**

1. 매개변수 확장: $AA
2. 산술 확장: $(( 1 + 2 ))
3. 명령 치환: $( echo "1.3 + 2.5" | bc )



### 조건문

#### if 문
```
if [ 조건 ]
then
  참일 경우
fi
```

조건을 쓸 때 [ ,]를 사용하는데, 앞에 공백을 넣어 명령이나 인수를 확실히 구분해주어야 한다.



`[10 -eq 10]` 라고 쓰면, `[10`가 명령 이름이 된다.

`[ 10 -eq 10]` 라고 쓰면, `10]` 가 하나의 인수가 된다.

`[ a=b ]`라고 쓰면, `a=b`가 하나의 인수로 인식되기 때문에 `[ a = b ]` 와 같이 써야한다.









#### if~else 문

```
if [ 조건 ]
then
  참일 경우
else
  거짓일 경우
fi
```

크게 조건문에 들어가는 비교 연산자는 `비교 연산자`와 `산술 연산자`, `파일 비교 연산자` 가 있다.



**비교연산자**

`"문자열" = "문자열" (두 문자열이 같으면 참)`

`"문자열" != "문자열"  (두 문자열이 같지 않으면 참)`

`-n "문자열" (문자열이 NULL이 아니면 참)`

`-z "문자열" (문자열이 NULL이면 참)`



**산술 연산자**

`수식 -eq 수식 (두 수식이 같으면 참)`

`수식 -ne 수식 (두 수식이 다르면 참)`

`수식 -gt 수식 (첫수식이 크면 참)`

`수식 -ge 수식 (첫수식이 크거나 같으면 참)`

`수식 -lt 수식 (첫수식이 작으면 참)`

`수식 -le 수식 (첫수식이 작거나 같으면 참)`

`!수식 (수식이 거짓이라면 참)`



**파일 비교 연산자**

`-d 파일이름 (파일이 디렉토리면 참)`

`-f 파일이름 (파일이 일반 파일이면 참)`

`-r 파일이름 (파일이 읽기 가능이면 참)`

`-w 파일이름 (파일이 쓰기 가능이면 참)`

`-x 파일이름 (파일이 실행 가능이면 참)`

`-e 파일이름 (파일이 존재하면 참)`

`-s 파일이름 (파일이 크기가 0이 아니면 참)`



여러 조건을 쓰고 싶을 때 `AND`, `OR`를 사용할 수 있다.

`AND (-a 또는 &&)`

`OR (-o 또는 ||)`



#### case-esac 문
if문은 참, 거짓 두 가지 경우에만 사용이 가능하지만, case는 여러 경우에 사용이 가능하다.
```
case $변수 in
  조건1)
    조건1의 내용;;
  조건2)
    조건2의 내용;;
  *)
    조건 1,2에 포함 안될때;;
esac
```
실행이 모두 끝났다면 `;;`를 붙여줘야 한다.



### std in, out을 나타내는 '-'



input에 사용되는 경우

```
cat test.txt
1
2
3

echo hello world | cat test.txt
1
2
3

echo hello world | cat - test.txt
hello world
1
2
3

echo hello world | cat test.txt -
1
2
3
hello world
```



output에 사용되는 경우

```
tar -czpvf - .
```

현재 경로에 있는 모든 파일들을 tar로 압축한다.



### 반복문

#### for~in 문
```
for 변수 in 값1, 값2, 값3 ...
do
  실행
done
```



값 대신 `..` 을 통해 범위로 잡을 수 있다.

```
for 변수 in {1..5}
do
  실행
done
```



변수 값을 통해 for을 동작시키고 싶을 땐 아래와 같다.

```
START=1
END=5
for (( i=$START; i<=$END; i++ ))
do
  실행
done
```



#### while 문

```
while [ 조건 ]
do
  실행
done
```
조건이 `참`이면 반복하며, 조건에 1이 들어가면 무한 반복한다. <br>
실행 취소를 원한다면 `ctrl`+`c`를 누른다.



#### until 문
while문과 용도가 거의 동일하지만, 조건이 `거짓`일 때만 반복한다.

반복문에서 `break`, `continue`를 통해 반복문을 빠져나오거나, 반복문의 조건식으로 다시 돌아갈 수 있다.



### 명령 종료 문자

shell script는 문장 끝에 반드시 ;를 붙일 필요는 없지만, 명령들을 한 줄에 연이어 쓰고 싶다면 ;을 붙어야 한다.



**에러 발생**

`for i in {1..3} do echo $i done`

**정상 동작**

`for i in {1..3}; do echo $i; done`



### 함수
```
함수이름 () {
  내용
}

함수이름
```

파라미터를 입력하고 싶다면, 함수이름 다음에 파라미터를 사용한다.



```
함수이름() {
  $1, $2 ..
}

함수이름 100, 200
```



함수에 return은 다른 프로그래밍 언어와는 다르게 연산 결과를 반환하는데 사용하지 않는다.

```
함수이름() {
  expr $1 + $2;
  return 5;
}
```

`echo $(함수이름 1 2)`를 통해 함수를 호출하면 결과는 3을 출력한다.

`echo $?` 를 통해 명령의 종료 상태 값인 5를 출력한다. (return 5;)



### 명령의 옵션

기본 명령들을 shell script에 쓸 때 주의해야 한다.

`grep -r '-n'`과 같이 쓰면 -n이라는 스트링을 찾는 것이 아니라, 명령으로 인식한다.

그렇기 때문에 `grep -r -- "-n"` 과 같이 -n이 명령이 아니라는 의미로 `--` 를 붙여주어야 한다.



### 그외 유용한 명령들

**eval** <br>
문자열을 명령문으로 인식하고 실행한다.
```
eval ls
```

> ls 명령이 실행된다.



**printf**
C언어의 printf()와 유사하게 사용할 수 있다.

```
printf "%5.2f %d \n\n" $var1 $var2
```



**$(명령어)**

$는 앞서 변수 값을 가져올 때 사용한다고 했지만, $와 괄호를 같이 사용하면 명령어로 실행할 수 있다.

```
$(명령어)
```



**set**

명령의 결과를 파라미터로 사용하고 싶을 때 사용한다.

```
set $(date)
echo $4
```

> `date` 명령을 수행의 결과 중 4번째 결과 값을 출력한다.



예제

```
List="aaa bbb ccc ddd"
set $List
echo $2
```

> List에 2번째인 'bbb'를 출력한다.



**shift**

쉘 스크립트는 최대 9개 명령형 인자를 받을 수 있기에 그 이상의 인자를 받기 위해 shift를 사용해야 한다.

인자는 $1 ~ $9 까지 사용이 가능하다. (1 ~ 9번)

즉, 10번 인자를 사용하기 위해, $1를 없애고 각 인자번호를 1씩 줄인다.



### 스크립트 예제

[Rack Awareness 설정](https://github.com/JaeGeunBang/JaeGeunBang.github.io/blob/master/_posts/2019-02-21-Rack%20Awareness%20%EC%84%A4%EC%A0%95.md)에 있는 topology.sh를 분석해보았다.

```bash
HADOOP_CONF=/home/hadoop/etc/hadoop

while [ $# -gt 0 ] ; do
  nodeArg=$1
  exec< ${HADOOP_CONF}/topology.data 
  result="" 
  while read line ; do
    ar=( $line ) 
    if [ "${ar[0]}" = "$nodeArg" ] ; then
      result="${ar[1]}"
    fi
  done 
  shift 
  if [ -z "$result" ] ; then
    echo -n "/default/rack "
  else
    echo -n "$result "
  fi
done
```
해당 .sh의 파라미터는 192.168.56.191, 192.168.56.192, 192.168.56.193, 192.168.56.194, 192.168,56,195라 한다.



`HADOOP_CONF`

- 하둡 디렉토리를 지정한다.

`while [$# -gt 0 ]; do`

- 파라미터의 수가 0보다 크면 반복문을 실행한다. 

`nodeArg=$1`

- 첫번째 파라미터인 192.168.56.191을 읽는다.

`exec< ${HADOOP_CONF}/topology.data`

- exec에 commnad 명령이 없으면, redirect 명령이 실행된다. 즉, topology.data를 input으로 읽는다.

`while read line ; do`

- read는 위에서 읽은 topology.data의 한 row를 읽어 line에 저장한다.
- line에 `192.168.56.191	dc_1/rack_1 `가 저장된다.

`if [ "${ar[0]}" = "$nodeArg" ] ; then`

- ar[0], nodeArg를 비교한다. 즉, 두 IP가 같은지 비교한다.
- 같다면 `result="${ar[1]}"` 를 수행한다. 즉, "dc_1/rack_1" 을 result 변수에 저장한다.

`shift`

- 첫번째 인자를 제거 후 왼쪽으로 민다(?). 즉, 기존 $1은 192.168.56.191 이였는데, shift를 수행한 후에 $1은 192.168.56.192가 된다. (다음 인자)

`if [ -z "$result" ] ; then`

- result 변수에 값이 없으면, `echo -n "/default/rack"`를 수행, 아니면 `echo -n "$result "`를 수행한다.



위 스크립트를 수행하면 최종 아래와 같이 출력된다.
```
dc_1/rack_1
dc_1/rack_2
dc_1/rack_2
dc_1/rack_1
dc_1/rack_1
```



## 참고

[Bash 입문자를 위한 핵심 요약 정리](https://blog.gaerae.com/2015/01/bash-hello-world.html)
> 예약 변수, 위치 매개 변수, 특수 매개변수, 매개 변수 확장 등 참고
