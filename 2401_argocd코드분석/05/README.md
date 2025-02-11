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

## [분석] DeltaFIFO 의 queue 에 값을 넣어주는 곳은?

* @TODO 찾아서 실험 진행

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
    * NewFilteredApplicationInformer 호출
* NewFilteredApplicationInformer 따라감
    * cache.NewSharedIndexInformer 호출
* cache.NewSharedIndexInformer 따라감
    * NewSharedIndexInformerWithOptions 호출
* NewSharedIndexInformerWithOptions 따라감
    * &sharedIndexInformer 호출
* 즉 appInformer 의 구현체는 sharedIndexInformer 이고 위에서 (*sharedIndexInformer).Run 에서 controller 만든 것과 연결됨
