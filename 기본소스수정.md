# cna demo Project

1. [msaez](http://msaez.io) 에서 모델링
2. IntelliJ 에서 소스 open
3. 프로그램 수정

### order 
- sync I/F를 위해 Feign 에서 사용할 다른 서비스의 URL을 환경별로 다르게 적용되도록 변수로 추가
    @resources/application.yml
    ```yaml
    profile: default
    api:
        url:
            delivery: http://localhost:8082
    ----
    profile: docker
    api:
        url:
            delivery: http://delivery:8082
    ```
* Feign 을 사용하는 java 파일에서 url 지정
   @external/CancellationService.java
   ```java
    @FeignClient(name="delivery", url="http://delivery:8080")
    -->
    @FeignClient(name="delivery", url="${api.url.delivery}") 
   ```

### delivery
 * Shipped Event Biz-Logic 구현
 @PolicyHandler
```java
@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    DeliveryRepository deliveryRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrdered_Ship(@Payload Ordered ordered){

        if(ordered.isMe()){
            Delivery delivery = new Delivery();
            delivery.setOrderId(ordered.getId());
            delivery.setStatus("SHIPPED");

            deliveryRepository.save(delivery);
        }
    }
}
```
  
### gateway
  - gateway 의 predicates 가 2개 이상인 경우 URL 사이에 , 추가
  ```yaml
    - Path=/deliveries/**/cancellations/** 
      --> 
    - Path=/deliveries/**,/cancelllations/**
  ```
  - 포트를 통한 서비스 실행 확인
  ```
  netstat -ano | findstr PID :808
  ```


## kafka
[참고 : apache-kafka installation on mac](https://medium.com/@Ankitthakur/apache-kafka-installation-on-mac-using-homebrew-a367cdefd273)
* install on mac
```sh
brew install kafka
```

* Start Zookeeper
```sh
# Windows
cd kafka/bin/windows
zookeeper-server-start.bat ../../config/zookeeper.properties

# Mac
zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties
```

* Start Kafka Server
```sh
# Windows
cd kafka/bin/windows
kafka-server-start.bat ../../config/server.properties

# Mac
kafka-server-start /usr/local/etc/kafka/server.properties
```
* Topic 이름 확인
```yaml
 @resources/application.yml
 spring.cloud.stream.bindings.event-out.destination
```
* Create Kafka Topic
```sh
kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic cnademo
```

* List of Topics
```sh
kafka-topics --zookeeper localhost:2181 --list
```

* Initialize Producer console
```
kafka-console-producer --broker-list localhost:9092 --topic cnademo
```

* Initialize Consumer console
```
kafka-console-consumer --bootstrap-server localhost:9092 --topic cnademo --from-beginning
```
