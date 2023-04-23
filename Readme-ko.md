# WebRtc, Coturn, SignalR and Blazor WebAssembly

Blazor WebAssembly는 피어 투 피어 애플리케이션을 개발할 수 있는 강력한 환경을 제공합니다. 
이 브라우저는 설치가 필요 없는 최고의 사용 편의성을 제공하며, WASM은 놀라울 정도로 빠릅니다.

샘플 애플리케이션에서는 자바스크립트 인터롭과 SignalR을 WebRTC 시그널링 채널로 사용하여 Blazor WebAssembly 애플리케이션에서 WebRTC를 통해 비디오를 스트리밍합니다. 
이 애플리케이션에는 두 가지 접근 방식이 사용됩니다.

첫째, Index.razor에서는 레이저 페이지의 요소와 직접 상호 작용하는 코드 비하인드 JS 파일을 사용합니다. 
이렇게 하면 인터롭을 통해 다른 js 호출에서 사용할 복잡한 객체를 Blazor를 통해 전달하지 않고도 비디오에서 직접 스트림을 설정할 수 있습니다. 
또한 Index.razor.js 샘플은 비동기/대기 대신 .then() 패턴을 훨씬 더 많이 사용합니다.

Google 샘플은 .then() 패턴을 사용하여 작성되었기 때문에 코드를 async/await으로 리팩터링하여 바로 변환하는 좋은 예로 삼고 싶었습니다. 

WebRTCService는 블레이저 크기에서 보다 서비스 지향적인 접근 방식을 보여주며, 자바스크립트에서는 보다 현대적인 비동기/대기 패턴을 사용합니다. 
함께 제공되는 자바스크립트는 결합된 WebRTCService 블레이저 클래스와 함께 있는 것이 아니라 wwwroot에 배치됩니다. 
WebRTCService는 program.cs의 서비스에 추가되고 Rtc.razor/Rtc.razor.js에서 사용됩니다.

이 클래스 쌍은 매우 간결합니다. 
거의 모든 무거운 작업은 WebRTCService에서 수행됩니다.
한 가지 참고 사항: WebRTCService에는 표준 .net 이벤트가 있습니다. 
이것은 약한 이벤트로 대체해야 합니다. 
저는 보통 PeanutButter.TinyEventAggregator를 사용합니다.


## WebRTC
WebRTC 호출을 하려면 자바스크립트에서 RTCPeerConnection(옵션)을 생성하고 이를 기반으로 CreateOffer(옵션)를 호출합니다. 
생성된 오퍼는 상대방에게 전송되어야 합니다. 
상대방은 오퍼를 받으면 RTCPeerConnection을 생성하고 그 위에 setRemoteDescription(오퍼)을 설정합니다. 
또한 발신자는 연결 후보를 전송하고, 수신자는 이를 RTCPeerConnection의 addIceCandidate(후보)에 전달합니다. 
그런 다음 createAnswer()를 호출하여 원래 당사자에게 다시 보냅니다. 
원래 상대방이 응답을 받으면 WebRTC 연결을 만드는 데 필요한 모든 것을 갖추게 됩니다.

## Signaling
오퍼, 답변, 연결 후보를 오퍼자와 답변자 간에 주고받아야 합니다. 
이를 '시그널링'이라고 합니다. 
저희의 경우, 블레이저 앱은 SignalR이 설치된 Asp.Net Core 웹서버에 연결되어 있습니다. 
단일 허브인 MessageHub는 매우 간단한 이름의 채널을 사용하여 모든 SignalR 트래픽을 양 당사자 간에 처리합니다.

## Coturn
WebRTC는 방화벽을 우회하기 위해 스턴/턴 서버가 필요합니다. 
Coturn을 설치하는 가장 쉬운 방법은 Docker가 설치된 Linux 서버에 설치하는 것입니다. 
명령 한 번으로 코턴 서버를 불러올 수 있는 명령은 다음과 같습니다. 

`docker run -d --network=host coturn/coturn -n --log-file=stdout --external-ip='$(detect-external-ip)' --relay-ip='$(detect-external-ip)'`

이 방법으로 코턴을 실행하면 포트 3478을 통해 서버에 스턴/턴을 설정합니다. 
스턴 서버는 일반적으로 자격 증명을 요구하지 않지만 턴 릴레이는 훨씬 더 비싸기 때문에 보통 비밀번호를 요구합니다. 
이는 프로덕션 환경에서 변경해야 하지만 기본값은 다음과 같습니다:

            username: 'username'
            credential: 'password'

일부 문서에는 's'가 붙은 '자격증명'으로 표시되어 있습니다. 틀렸습니다.

            credentials: 'password'
 
