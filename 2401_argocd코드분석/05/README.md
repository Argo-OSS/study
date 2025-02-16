# 2401 ArgoCD 코드분석 스터디 - 5회차

* ArgoCD 기준 Commit : 96d0226a4963d9639aea81ec1d3a310fed390133

## [Recap] UI 에서 변경 감지 전달받는 프로세스

* @TODO 나중에 정리할 것

### [실험] Deployment 를 삭제했을 때 Broadcaster (broadcasterHandler) 의 어떤 메소드를 호출할까?

* 다음 3개 메소드에 break point 설정
    * server/application/broadcaster.go
    * (*broadcasterHandler).OnAdd (L79)
    * (*broadcasterHandler).OnUpdate (L85)
    * (*broadcasterHandler).OnDelete (L91)
* argocd-example-apps 의 helm-guestbook 배포
* k9s 로 접근해서 배포한 앱의 deployment 삭제
* (*broadcasterHandler).OnUpdate 에서 중지
    * app.ObjectMeta 가 helm-guestbook 인 것 확인
    * app.Status.Sync 가 OutOfSync 인 것 확인
    * app.Status.Health 가 Missing 인 것 확인
* (*broadcasterHandler).OnUpdate 는 (*processorListener).run() 에서 호출
    * k8s.io/client-go/tools/cache/shared_informer.go:L976
    * (*processorListener).nextCh 에서 값을 전달받아서 전파

## [분석] (*processorListener).nextCh 에 값을 넣어주는 곳은?

* (*processorListener).pop 메소드
* (*processorListener).addCh 에서 값을 전달받아 (*processorListener).nextCh 에 전달
* (*processorListener).pendingNotifications 를 버퍼로 사용해 배압 조절

### [실험] (*processorListener).pop 무한 루프의 실제 동작 확인

* 다음 메소드 내부에 break point 설정
    * vendor/k8s.io/client-go/tools/cache/shared_informer.go:L943
    * (*processorListener).pop 메소드 내부 무한 루프
* argocd-example-apps 의 helm-guestbook 배포
* k9s 로 접근해서 배포한 앱의 deployment 삭제
* 최초에 L951 에서 대기중인 코드가 풀리면서 notification, nextCh 이 세팅됨 (break point 디버거 붙기 전 이미 해당 라인에서 block 된 상태)
    * notification 의 oldObj, newObj 로 helm-guestbook 의 변경 내용인 것 확인
* 다음 루프에서 첫 번째 select 에 걸리면서 변경 전파

## [분석] (*processorListener).addCh 에 값을 넣어주는 곳은?

* (*processorListener).add 메소드

### [실험] 누가 (*processorListener).add 메소드를 호출할까?

* (*processorListener).add 메소드 내부에 break point 설정
    * vendor/k8s.io/client-go/tools/cache/shared_informer.go:L930
* argocd-example-apps 의 helm-guestbook 배포
* k9s 로 접근해서 배포한 앱의 deployment 삭제
* (*sharedProcessor).distribute 에서 호출
    * obj 가 oldObj, newObj 가지고 있음
* 위 메소드는 (*sharedIndexInformer).OnUpdate 에서 호출
    * old, new 받아서 oldObj, newObj 로 전달
* cache.processDeltas 에서 호출
    * delta 인자로 변경 목록 받음
        * 여기에 newObj 있음
    * delta 의 Object (newObj) 를 가지고 clientState 에서 oldObj 조회 -> 이걸 전달
* (*sharedIndexInformer).HandleDeltas 에서 호출
    * delta 를 받아서 cache.processDeltas 에 전달
* (*DeltaFIFO).Pop 에서 호출
    * 내부 큐에서 item 을 하나씩 꺼내서 인자로 받은 PopProcessFunc 에 전달 -> 큐에 item 넣는 쪽 확인 필요
* (*controller).processLoop 에서 호출
    * Pop 트리거해서 큐 처리 프로세스 시작/에러시 재시작
    * Pop 에 전달한 함수는 PopProcessFunc(c.config.Process) -> 이게 결국 (*sharedIndexInformer).HandleDeltas

## [분석] controller 와 appInformer 의 연결

* controller 의 생성 과정을 역추적
* controller 를 생성하는 곳
  * vendor/k8s.io/client-go/tools/cache/controller.go:L124
  * func New(c *Config) Controller
  * Controller 는 인터페이스, 실제 구조체는 controller
* New 호출하는 곳
  * vendor/k8s.io/client-go/tools/cache/shared_informer.go:L490
  * (*sharedIndexInformer).Run
    * 여기서 (*sharedIndexInformer).HandleDeltas 가 c.config.Process 에 연결되는 걸 확인할 수 있음
* (*sharedIndexInformer).Run 호출하는 곳
  * server/server.go:L525
  * (*ArgoCDServer).Init
* (*ArgoCDServer).appInformer 만드는 곳
  * server/server.go:L301
  * appFactory.Argoproj().V1alpha1().Applications().Informer() 를 통해 appInformer 생성
* 마지막 Informer() 따라감
  * f.factory.InformerFor(&applicationv1alpha1.Application{}, f.defaultInformer) 호출 확인
* f.factory.InformerFor 따라감
  * 인자로 받은 newFunc 를 호출해서 informer 생성
* 다시 f.factory.InformerFor(&applicationv1alpha1.Application{}, f.defaultInformer) 호출하는 곳 보면 f.defaultInformer 가 newFunc
  임
* f.defaultInformer 따라감
  * NewFilteredApplicationInformer 호출 -> 여기서 cache.ListWatch 생성하면서 WatchFunc 만들어서 넘겨줌, 아래 분석 중간에 연결됨
* NewFilteredApplicationInformer 따라감
  * cache.NewSharedIndexInformer 호출
* cache.NewSharedIndexInformer 따라감
  * NewSharedIndexInformerWithOptions 호출
* NewSharedIndexInformerWithOptions 따라감
  * &sharedIndexInformer 호출
* 즉 appInformer 의 구현체는 sharedIndexInformer 이고 위에서 (*sharedIndexInformer).Run 에서 controller 만든 것과 연결됨

## [분석] DeltaFIFO 의 queue 에 값을 넣어주는 곳은?

* 아래 두 곳 중 하나
    * vendor/k8s.io/client-go/tools/cache/delta_fifo.go:L393
    * vendor/k8s.io/client-go/tools/cache/delta_fifo.go:L481
    * L393 먼저 break point 이후 Deployment 삭제 실험 -> 안 걸림
    * L481 에 break point 이후 Deployment 삭제 실험 -> 걸림
    * obj 가 그동안 본 newObj
* (*DeltaFIFO).queueActionInternalLocked 가 queue 에 넣어주고 있음
* (*DeltaFIFO).queueActionLocked 에서 호출
* (*DeltaFIFO).Update 에서 호출
* vendor/k8s.io/client-go/tools/cache/reflector.go:L831 의 handleAnyWatch 함수 내부에서 호출
    * event 구조체가 변경 데이터 들고있음
    * event 구조체는 w.ResultChan() 채널에서 받아옴
    * w 는 함수 인자로 받음
    * 받아온 w 의 인터페이스는 watch.Interface 이고 실제 타입은 *watch.StreamWater 인 것 확인
* handleWatch 함수에서 호출
    * w 는 마찬가지로 함수 인자로 받음
    * 여기서도 *watch.StreamWater 인 것 확인
* cache.(*Reflector).watch() 에서 호출
    * w 는 마찬가지로 인자로 받음
    * 여기서 handleWatch 에 넘겨주는 값 *watch.StreamWater 인 것 확인
* cache.(*Reflector).watchWithResync() 에서 호출
    * w 는 마찬가지로 인자로 받음
    * 여기서는 w 가 nil 임
    * 직전 cache.(*Reflector).watch() 내부를 다시 살펴봄
* cache.(*Reflector).watch() 내부에서 인자로 받은 w 가 nil 일 때 cache.(*Reflector).listerWatcher.Watch(options) 으로 w 만드는 것 확인
    * 이 때 cache.(*Reflector).listerWatcher 는 cache.ListWatcher 인터페이스에 실제 타입은 *cache.ListWatch 인 것 확인
* cache 패키지의 ListWatch 구조체의 Watch 메소드로 이동
    * vendor/k8s.io/client-go/tools/cache/listwatch.go:L114
    * ListWatch 의 WatchFunc 사용하는 것 확인 -> 위 분석 내용에서 봤던 함수
* event 를 쏴주는 w.ResultChan() 채널을 client.ArgoprojV1alpha1().Applications(namespace).Watch(context.TODO(), options) 여기서 찾아봄
    * 인터페이스의 구현체 따라가면 vendor/k8s.io/client-go/gentype/type.go:L211
    * c.client 의 Watch() 결과를 리턴
* c.client 의 Watch() 내부
    * vendor/k8s.io/client-go/rest/request.go:L706
    * r.newStreamWatcher(resp) 를 리턴
* (*Request).newStreamWatcher 내부
    * watch.NewStreamWatcher(...) 리턴
* vendor/k8s.io/apimachinery/pkg/watch/streamwatcher.go:L62 에서 &StreamWatcher{} 리턴
* (*StreamWatcher).ResultChan() 은 vendor/k8s.io/apimachinery/pkg/watch/streamwatcher.go:L81 에 위치
    * (*StreamWatcher).result (채널) 를 반환
* (*StreamWatcher).result 에 값을 넣어주는 곳은?
    * vendor/k8s.io/apimachinery/pkg/watch/streamwatcher.go:L118
    * vendor/k8s.io/apimachinery/pkg/watch/streamwatcher.go:L130
    * L118 은 에러 이벤트 전달, L130 은 실제 이벤트 전달 
    * L130 이 위 handleAnyWatch 에서 받아오는 event 를 넣어주는 곳 -> 실험으로 확인

### [실험] (*StreamWatcher).result 에 값 넣어주는 곳 확인

* vendor/k8s.io/apimachinery/pkg/watch/streamwatcher.go:L130 에 break point
* argocd-example-apps 의 helm-guestbook 배포
* k9s 로 접근해서 배포한 앱의 deployment 삭제
* break point 에서 action, obj 가 helm-guestbook 에 관한 내용인 것 확인

## [분석] (*StreamWatcher).result 에 넣는 값은 어디서 가져오는지?

* vendor/k8s.io/apimachinery/pkg/watch/streamwatcher.go:105
    * 무한 루프 내부에서 
    * (*StreamWatcher).source.Decode() 를 호출해서 값 가져옴
    * vendor/k8s.io/apimachinery/pkg/watch/streamwatcher.go:106 에 break point 걸고 확인 가능
        * 왜 106 라인인가?
        * 이미 (*StreamWatcher).source.Decode() 는 호출된 상태로 blocking 되어있는 상태라서?
* (*StreamWatcher).source 를 추적하기 위해 다시 watch.NewStreamWatcher 로 돌아오면 첫 번째 인자 d Decoder 임
* watch.NewStreamWatcher 호출부 -> (*Request).newStreamWatcher (vendor/k8s.io/client-go/rest/request.go:L926)
* restclientwatch.NewDecoder(watchEventDecoder, objectDecoder) 가 첫 번째 인자
* restclientwatch.NewDecoder 내부에서 Decoder 생성, 반환 (vendor/k8s.io/client-go/rest/watch/decoder.go:L39)
* (*Decoder).Decode 메소드 확인 (vendor/k8s.io/client-go/rest/watch/decoder.go:L47)
    * (*Decoder).decoder.Decode() 에서 데이터 가져옴
* (*Decoder).decoder 추적
    * restclientwatch.NewDecoder 의 첫 번째 인자
    * vendor/k8s.io/client-go/rest/request.go:L927 에서 다시 확인해보면 watchEventDecoder 임
* watchEventDecoder 는 streaming.NewDecoder(frameReader, streamingSerializer) 
* vendor/k8s.io/apimachinery/pkg/runtime/serializer/streaming/streaming.go:L62 에서 생성
* (*decoder).Decode() 메소드 추적 (vendor/k8s.io/apimachinery/pkg/runtime/serializer/streaming/streaming.go:L74)
    * vendor/k8s.io/apimachinery/pkg/runtime/serializer/streaming/streaming.go:L77 의 d.reader.Read 로 데이터 읽어옴
    * d.reader 추적
* 다시 NewDecoder (vendor/k8s.io/apimachinery/pkg/runtime/serializer/streaming/streaming.go:L62) 에 가면 첫 인자로 받은 r io.ReadCloser 임
* NewDecoder 호출부로 이동 -> vendor/k8s.io/client-go/rest/request.go:L924
* 첫 번째 인자는 frameReader := framer.NewFrameReader(resp.Body)
    * frameReader 는 resp.Body 의 데이터를 가공하는 역할만 하는 것으로 추정
    * resp 추적
    * (*Request).newStreamWatcher 의 인자로 받아옴
    * (*Request).newStreamWatcher 호출부로 이동 -> vendor/k8s.io/client-go/rest/request.go:L741
    * resp 는 (*Request).newHTTPRequest(ctx) (vendor/k8s.io/client-go/rest/request.go:L733) 의 실행 결과
* (*Request).newHTTPRequest(ctx) 내부에서 (*Request).URL().String() 으로 요청 보내는 것 확인
    * (*Request).URL() 내부를 보면 Request 의 필드에 있는 정보로 URL 구성중
    * 이 데이터는 pkg/client/informers/externalversions/application/v1alpha1/application.go:L55 에서 빌더 패턴으로 넣어준 것으로 추측
    * 위 라인에서 Watch 호출 직전의 Applications 메소드 분석
    * pkg/client/clientset/versioned/typed/application/v1alpha1/application_client.go:L29 에서 newApplications 호출
    * pkg/client/clientset/versioned/typed/application/v1alpha1/application.go:L42 에서 applications 생성
    * 여기서 resources 로 applications 지정하는 것 확인 -> 실험에서 데이터 찾는 키 

### [실험] (*Request).URL().String() 확인

* Makefile 에서 start-local 명령어 찾아서 mod-vendor-local 잠시 disable
* vendor/k8s.io/client-go/rest/request.go:L727 아래에서 생성한 URL 출력 (fmt.Printf("KHLTEST:::%s\n", url))
* 다시 make start-local ARGOCD_GPG_ENABLED=false 실행
* 실행 로그에서 
    * api-server 의 로그 중
    * KHLTEST::: 로 시작하는 로그 중
    * ArgoprojV1alpha1 키워드 포함하는 로그 중
    * namespaces/argocd 포함하는 로그 중
    * applications 포함하는 로그 찾기
    * https://127.0.0.1:62963/apis/argoproj.io/v1alpha1/namespaces/argocd/applications?allowWatchBookmarks=true&resourceVersion=6760&timeout=7m25s&timeoutSeconds=445&watch=true
* 위 API 호출 결과가 (*StreamWatcher).result 를 통해 전파됨 
* 아마도 k8s API?
