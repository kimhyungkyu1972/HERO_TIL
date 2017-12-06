# 스파크 스트리밍

---

여태까지는 저장된 데이터를 가져다가 썼다. 그렇다면 이제는 (거의) 실시간으로 흐르는 데이터를 찾아 처리해보자.

스트림에는 다양한 뜻이 있는데, 대강 여기에서는 데이터를 어디에 쌓지 않고 가능한 한은 바로바로 처리하는 것을 생각하면 된다. 여태 했던 것은 **배치 처리** 라고 부른다.

스파크 스트림 데이터 처리는 **DStream**  이라는 것으로 변환해서 처리되지만, 내부적으론 RDD 로 변환되어 쓰인다. 즉, 내부적으로는 스파크 스트리밍이 데이터를 받아서 배치 처리로 나눠서 처리하며, 결과는 배치 형태로 나오는 것이다.

흘러가는 스트림 데이터는 아래와 같은 데이터 소스들에서 가져오는게 좋다.

- kafka
- flume
- kinesis
- twtitter
- mqtt
- zeromq

![](https://imgur.com/a/vKf4q)

파이썬 api에서 모든 버전을 지원하지는 않는다는 것은 유념해 두도록 한다. spark api docs 등을 참고해서 버전 서포트를 꼭 확인하자. kafka 등은 2.1.1 버전부터 지원한다.

> Kafka: Spark Streaming 2.2.0 is compatible with Kafka broker versions 0.8.2.1 or higher. See the Kafka Integration Guide for more details.

> Flume: Spark Streaming 2.2.0 is compatible with Flume 1.6.0. See the Flume Integration Guide for more details.

> Kinesis: Spark Streaming 2.2.0 is compatible with Kinesis Client Library 1.2.1. See the Kinesis Integration Guide for more details.


## structured streaming

스트리밍 처리를 시작하기 전에, [structrured streaming](http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) 을 먼저 참고한다. 이 기능은 2.1.1까지는 알파, 2.1.2인가부터는 베타버전이었으나 2.2.0부터는 이제 정 기능이 되었다.

> Structured Streaming is a scalable and fault-tolerant stream processing engine built on the Spark SQL engine. 

안에서 python 코드를 확인해볼 수 있으며, 스트리밍 처리를 위해서는 한번 확인해보는것이 좋다. 

structure stream이란 데이터 스트림을 잘게 쪼개 배치처리를 하는 것인데, unbound table 이란 곳에 저장된다. *Unbound table* 이 뭐냐면, 어느 정도의 크기를 가지고 있는지 정해지지 않은 테이블을 뜻한다. 스트림 처리를 할 땐 데이터 크기가 얼마나 될 지 예상할수가 없기 때문이다.

트리거마다 새 테이블 인풋 쿼리가 발생한다고 생각하면 된다. 그래서 바로바로 결과가 바뀌는것을 확인 가능하다.

## spark streaming

원래 하려던 스파크 스트리밍(좀 더 예전에 나왔다.) 을 살펴보도록 하자. 

```
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

# Create a local StreamingContext with two working thread and batch interval of 1 second
sc = SparkContext("local[2]", "NetworkWordCount")
ssc = StreamingContext(sc, 1)
```

context는 스파크에 접근하는 가장 기본적인 방법이다. 물론 스파크 스트리밍에서도 활용할 수 있다.

---

### 예제

스파크 스트리밍을 사용한 소켓 커넥션에서 스트림을 받아오는 예제를 구축한다.

```
import findspark

findspark.init("불러올 스파크 주소")

from pyspark import SparkContext
from pyspark.streaming import StreamingContext

sc = SparkContext("local[2]", "예제")
# local[2]? : 두 개의 로컬에서 작업한다는 얘기

ssc = StreamingContext(sc, 1)
# 1? : 인터벌은 1초로 한다는 얘기

lines = ssc.socketTextStream("localhost", "포트번호")
# Dstream을 만들어냄
# 연결할 주소, 포트번호를 준다. 보통 로컬에서 하면 9999로 그냥 넘기면 될 듯 하다.

words = lines.flatMap(lambda line : line.split(" "))
# lines 을 맵
# lambda로 " "(띄어쓰기) 를 기준으로 나눴다.

pairs = words.map(lambda word : (word, 1))
# 맵을 tuple 로 만들었다.

word_counts = pairs.reduceByKey(lambda num1, num2 : num1+num2)
# 글자별로 나타난 횟수를 더해줬다.
# 여기서는 reduceByKey 므로 키별 밸류를 더하고 있는 것이다.

word_counts.pprint()
ssc.start()
```

cmd창 하나를 켜고 `nc -lk 9999` 를 입력 후 아무 단어나 입력해서, 위 프로그램이 가동하면서 자동으로 내가 입력한 단어의 수를 세는 것을 확인해볼 수 있다.

---

### 트위터 스트리밍 받아오기

흘러가는 빠른 데이터의 대명사인 sns에서 스트리밍 데이터를 받아보자.

[apps.twitter.com](apps.twitter.com)  부터 가서 트위터 계정을 만든 후, 로그인한다.
그리고 새 앱을 만들어서 api key 를 발급받는다.

발급받을 수 있는 키들은 네 가지다. 컨슈머 키, 컨슈머 시크릿 키, 액세스 토큰, 액세스 시크릿 토큰 키이다.
**이 키들을 깃허브에 올리거나 코드안에 넣지 않도록 주의하자.**

그리고 pip로 `tweepy` 등을 받아서 twitter 를 쉽게 다루도록 하면 훨씬! 편하다.
`matplotlib` 이나 `seaborn` 을 받아서 비쥬얼라이즈에 사용해도 좋다.(우리는 제플린 내에서 사용한다)

tweepy를 사용하겠다.

```
import tweepy
from tweept import OAuthHandler, Stream
from tweepy.streaming import StreamListener
import socket
import json


consumer_key = ""
consumer_secret = ""
access_token = ""
access_secret = ""
# 자신의 값을 넣는다(사실 이렇게 하지말고 os. 에서 불러오는게 낫다. 깃허브에 키 올리면 매일 메일오고 키도 다시 취소된다. 특히 aws 할때 제발 키를 깃허브에 그만 올리자)

class TweetListener(StreamListener) :
    def __init__ (self, csocket) :
        self.client_socket = csocket
    def on_data(self, data) :
        try :
            msg = json.loads(data)
            print(msg['text'].encode('utf-8'))
            self.client_socket.send(msg['text'].encode('utf-8'))
            return True
        except BaseException as e :
            print("[ERROR] ", e)
            return True
        
    def on_error(self, status) :
        print(status)
        return True


def sendData(c_socket) :
    auth = OAuthHandler(consumer_key, consumer_secret)
    auth.set_access_token(access_token, access_secret)
    # 어느 아이디로 할건지
    
    twitter_stream = Stream(auth.TweetListener(c_socket))
    twitter_stream.filter(track=['인터파크'])
    # 인터파크라는 단어가 있는 트윗을 필터링 해보자.

if __name__ == "__main__" :
    s = socket.socket()
    host = "127.0.0.1"
    port = 5555
    s.bind((host,port))
    print("연5555결")
    # 우선 연결했다.
    
    s.listen(5)
    # 5초 기다리고
    c, addr = s.accept()
    sendData(c)
    # 실행시 연5555결이 뜨면서 기다리고 있음
        
```

만약 데이터 양이 그리 많지 않다면 이렇게 끝내도 되지만, 스파크를 더해 본다면 어떨까?








