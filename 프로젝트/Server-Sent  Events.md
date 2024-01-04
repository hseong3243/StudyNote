## 개요
---
Server-Sent Evenets, SSE는 [[HTTP]] 연결을 이용하여 서버가 클라이언트로 데이터를 전송하는 단방향 통신 기술입니다.
이는 메시지 업데이트 또는 지속적인 데이터 스트림을 브라우저 클라이언트에 전송하는데 사용됩니다.
[[웹소켓]]과의 주요한 차이점은 다음과 같습니다.
- 서버 -> 클라이언트 단반향 연결입니다.
- 별도의 프로토콜이 아닌 HTTP를 그대로 사용합니다. 따라서 별도의 프로토콜에 대한 학습이 필요치 않습니다.

클라이언트에서는 이를 사용하기 위해 `EventSource` 객체를 생성하여 이벤트 리스너를 등록하면됩니다. 
```javascript
const sse = new EventSource("http://localhost:8080/api/v1/notifications/connect");

sse.addEventListener('delivery', e => {  
  const {data: receivedCount} = e;  
  console.log("배달 메시지: ", receivedCount);  
})
```
서버 측에서는 `text/event-stream` MIME 타입을 이용하여 다음의 유형으로 메시지를 전송합니다.


## 구현
---
스프링 프레임워크는 `SseEmitter` 클래스를 제공하여 손쉽게 `SSE`를 구현할 수 있는 방법을 제공합니다. 다음은 클라이언트-서버 연결을 위한 api를 작성한 것입니다.
```java
@RestController  
@RequiredArgsConstructor  
@RequestMapping("/api/v1/notifications")  
public class NotificationController {  
  
    private final NotificationService notificationService;  
  
    @GetMapping(value = "/connect", produces = "text/event-stream")  
    public ResponseEntity<SseEmitter> sseConnection(  
        @RequestHeader(value = "Last-Event-ID", required = false, defaultValue = "") String lastEventId,  
        @LoginUser Long userId) {  
  
        ConnectNotificationCommand connectNotificationCommand  
            = ConnectNotificationCommand.of(userId, lastEventId);  
        SseEmitter sseEmitter = notificationService.connectNotification(connectNotificationCommand);  
        return ResponseEntity.ok(sseEmitter);  
    }  
}
```
- `Last-Event-ID`: 이벤트 소스의 마지막 이벤트 ID값
- `userId`: 로그인한 사용자의 ID값입니다. 이를 이용하여 이벤트 ID를 만들어줄 것입니다.

