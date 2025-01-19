# ArgoCD 코드분석 03회차

## 3회차 목표
* ArgoCD로 관리되는 리소스들에 변경이 생겼을 때 ArgoCD는 어떻게 이것을 감지하고 UI로 보여주는지 확인.

## 코드 확인 전 예상되는 확인 포인트
1. ArgoCD에서 관리 주체는 어떻게 정의되고 동작하는가?
- ArgoCD가 관리하는 대상 앱 및 리소스의 주체는 무엇인지, 그리고 관리 범위는 어떻게 설정되는지.
- 리소스의 관리 주체가 변경되거나 중복 관리 시 어떤 로직이 적용되는지.
2. ArgoCD는 리소스 상태를 어디에, 그리고 어떤 방식으로 저장하는가?
- 상태 정보를 저장하는 데이터 구조 및 흐름 분석 (예: Redis, Kubernetes CRD 등).
- 상태 정보가 실시간으로 동기화될까?
3. 변경 감지 방식과 많은 앱을 관리하기 위한 최적화는 어떻게 이루어지는가?
- ArgoCD가 Kubernetes API를 사용해 변경 이벤트를 감지하는 방법.
- 모든 이벤트를 수신하거나 필터링하는지, 이를 통해 성능 저하를 방지하는 기법은 무엇인지.
- 변경 이벤트를 기반으로 UI 업데이트가 발생하는 전체 흐름 분석.

## application의 리소스 
애플리케이션의 리소스 정보를 보면 상태 값에 Argo 애플리케이션을 통해 관리되고 있는 리소스 정보를 확인할 수 있다. 
이말은 application controller가 상태 값을 업데이트 해주는 것으로 이해가 되는데 
이 정보를 UI에서 어떻게 가져오고 처리하고 있는지를 확인해보고자 한다. 

application 리소스 예시 
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  creationTimestamp: "2025-01-14T15:05:00Z"
  generation: 4
  name: app
  namespace: argocd
  resourceVersion: "2788"
  uid: af626036-51e8-4365-8535-5a0a23746d87
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    path: kustomize-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
status:
  controllerNamespace: argocd
  health:
    status: Missing
  reconciledAt: "2025-01-14T15:05:57Z"
  resources: # 이 앱이 관리하는 app에 대한 정보가 담겨있음 
  - health:
      status: Missing
    kind: Service
    name: kustomize-guestbook-ui
    namespace: default
    status: OutOfSync
    version: v1
  - group: apps
    health:
      status: Missing
    kind: Deployment
    name: kustomize-guestbook-ui
    namespace: default
    status: OutOfSync
    version: v1
  sourceType: Kustomize
  summary: {}
  sync:
    comparedTo:
      destination:
        namespace: default
        server: https://kubernetes.default.svc
      source:
        path: kustomize-guestbook
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
    revision: d7927a27b4533926b7d86b5f249cd9ebe7625e90
    status: OutOfSync
```

먼저 application의 리스트를 가져와서 변경 점이 있는지 확인하는 것 같은 부분은 다음과 같다.
```typescript
// https://github.com/argoproj/argo-cd/blob/v2.13.3/ui/src/app/applications/components/applications-list/applications-list.tsx#L54
function loadApplications(projects: string[], appNamespace: string): Observable<models.Application[]> {
    return from(services.applications.list(projects, {appNamespace, fields: APP_LIST_FIELDS})).pipe(
        mergeMap(applicationsList => {
            const applications = applicationsList.items;
            return merge(
                from([applications]),
                services.applications
                    // application 리스트의 resourceVersion의 변경을 감시함 
                    .watch({projects, resourceVersion: applicationsList.metadata.resourceVersion}, {fields: APP_WATCH_FIELDS}) 
                    .pipe(repeat())
                    .pipe(retryWhen(errors => errors.pipe(delay(WATCH_RETRY_TIMEOUT))))
                    // batch events to avoid constant re-rendering and improve UI performance
                    .pipe(bufferTime(EVENTS_BUFFER_TIMEOUT))
                    .pipe(
                        map(appChanges => {
                            appChanges.forEach(appChange => {
                                const index = applications.findIndex(item => AppUtils.appInstanceName(item) === AppUtils.appInstanceName(appChange.application));
                                switch (appChange.type) {
                                    // App의 변경 상태가 삭제인 경우 처리를 함
                                    case 'DELETED':
                                        if (index > -1) {
                                            applications.splice(index, 1);
                                        }
                                        break;
                                    default:
                                        if (index > -1) {
                                            applications[index] = appChange.application;
                                        } else {
                                            applications.unshift(appChange.application);
                                        }
                                        break;
                                }
                            });
                            return {applications, updated: appChanges.length > 0};
                        })
                    )
                    .pipe(filter(item => item.updated))
                    .pipe(map(item => item.applications))
            );
        })
    );
}
```

```yaml
# kubectl get deployment -o yaml
apiVersion: v1
items: []
kind: List
metadata:
  resourceVersion: "" ## List 조회에도 이렇게 리소스 버전정보가 들어있음, 근데 다 빈칸이던데.. 
```

좀더 위에 보면 사전에 감시할 필드에 대한 정의가 있음
```typescript
// The applications list/watch API supports only selected set of fields.
// Make sure to register any new fields in the `appFields` map of `pkg/apiclient/application/forwarder_overwrite.go`.
const APP_FIELDS = [
    'metadata.name',
    'metadata.namespace',
    'metadata.annotations',
    'metadata.labels',
    'metadata.creationTimestamp',
    'metadata.deletionTimestamp',
    'spec',
    'operation.sync',
    'status.sync.status',
    'status.sync.revision',
    'status.health',
    'status.operationState.phase',
    'status.operationState.finishedAt',
    'status.operationState.operation.sync',
    'status.summary',
    'status.resources'
];
const APP_LIST_FIELDS = ['metadata.resourceVersion', ...APP_FIELDS.map(field => `items.${field}`)];
const APP_WATCH_FIELDS = ['result.type', ...APP_FIELDS.map(field => `result.application.${field}`)];

```

그럼 어떻게 서비스의 리스트를 갖고오는지 좀더 확인해보도록 한다.
ApplicationsService에서 애플리케이션에 대한 생성/삭제/조회 등등의 역할을 하고있다. (동기화를 특정시간에만 가능하게 하는 동기화윈도우 기능도있엇네.!?)

그중에서도 위에서 나왔던 watch 코드에 대해서 좀더 살펴 보면 서버로부터 실시간 이벤트를 수신하고 있는것을 알 수 있다. 
```typescript
// https://github.com/argoproj/argo-cd/blob/v2.13.3/ui/src/app/shared/services/applications-service.ts#L170
    public watch(query?: {name?: string; resourceVersion?: string; projects?: string[]; appNamespace?: string}, options?: QueryOptions): Observable<models.ApplicationWatchEvent> {
        const search = new URLSearchParams();
        if (query) {
            if (query.name) {
                search.set('name', query.name);
            }
            if (query.resourceVersion) {
                search.set('resourceVersion', query.resourceVersion);
            }
            if (query.appNamespace) {
                search.set('appNamespace', query.appNamespace);
            }
        }
        if (options) {
            const searchOptions = optionsToSearch(options);
            search.set('fields', searchOptions.fields);
            search.set('selector', searchOptions.selector);
            search.set('appNamespace', searchOptions.appNamespace);
            query?.projects?.forEach(project => search.append('projects', project));
        }
        const searchStr = search.toString();
        const url = `/stream/applications${(searchStr && '?' + searchStr) || ''}`;
        return requests
            .loadEventSource(url)
            .pipe(repeat())
            .pipe(retry())
            .pipe(map(data => JSON.parse(data).result as models.ApplicationWatchEvent))
            .pipe(
                map(watchEvent => {
                    watchEvent.application = this.parseAppFields(watchEvent.application);
                    return watchEvent;
                });
            );
    }
```

그럼 이제 watch라는 함수가 /stream/applications ~~로 시작되는 경로로 호출을 하는것을 확인했고
그 경로에 대한 처리를 찾아보면 아래의 코드에서 확인이 가능하다. 

```golang
// https://github.com/argoproj/argo-cd/blob/v2.13.3/server/application/application.go#L1162

// <참고용> //server/applicaion/application.proto
// message ApplicationQuery {
// 	// the application's name
//	optional string name = 1;
//	// forces application reconciliation if set to 'hard'
//	optional string refresh = 2;
//	// the project names to restrict returned list applications
//	repeated string projects = 3;
//	// when specified with a watch call, shows changes that occur after that particular version of a resource.
//	optional string resourceVersion = 4;
//	// the selector to restrict returned list to applications only with matched labels
//	optional string selector = 5;
//	// the repoURL to restrict returned list applications
//	optional string repo = 6;
//	// the application's namespace
//	optional string appNamespace = 7;
//	// the project names to restrict returned list applications (legacy name for backwards-compatibility)
//	repeated string project = 8;
//}

// pkg/apis/application/v1alpha1/types.go
// ApplicationWatchEvent contains information about application change.
// ApplicationWatchEvent struct {
//	Type watch.EventType `json:"type" protobuf:"bytes,1,opt,name=type,casttype=k8s.io/apimachinery/pkg/watch.EventType"`
//
//	// Application is:
//	//  * If Type is Added or Modified: the new state of the object.
//	//  * If Type is Deleted: the state of the object immediately before deletion.
//	//  * If Type is Error: *api.Status is recommended; other types may make sense
//	//    depending on context.
//	Application Application `json:"application" protobuf:"bytes,2,opt,name=application"`
//}


func (s *Server) Watch(q *application.ApplicationQuery, ws application.ApplicationService_WatchServer) error {
	appName := q.GetName()  // 애플리케이션 이름
	appNs := s.appNamespaceOrDefault(q.GetAppNamespace())  // 애플리케이션 네임스페이스
	logCtx := log.NewEntry(log.New())
	if q.Name != nil {
		logCtx = logCtx.WithField("application", *q.Name)
	}
	projects := map[string]bool{}
	for _, project := range getProjectsFromApplicationQuery(*q) {
		projects[project] = true
	}
	claims := ws.Context().Value("claims")
	selector, err := labels.Parse(q.GetSelector())
	if err != nil {
		return fmt.Errorf("error parsing labels with selectors: %w", err)
	}
	minVersion := 0 // 리소스 버전이 있는지 조회 
	if q.GetResourceVersion() != "" {
		if minVersion, err = strconv.Atoi(q.GetResourceVersion()); err != nil { 
			minVersion = 0
		}
	}

	// sendIfPermitted is a helper to send the application to the client's streaming channel if the
	// caller has RBAC privileges permissions to view it RBAC 검사
	sendIfPermitted := func(a appv1.Application, eventType watch.EventType) {
		permitted := s.isApplicationPermitted(selector, minVersion, claims, appName, appNs, projects, a)
		if !permitted {
			return
		}
		s.inferResourcesStatusHealth(&a)
        // client에게 이벤트를 전송
		err := ws.Send(&appv1.ApplicationWatchEvent{
			Type:        eventType,
			Application: a,
		})
		if err != nil {
			logCtx.Warnf("Unable to send stream message: %v", err)
			return
		}
	}
	// 애플리케이션 변경 이벤트를 저장하는 채널
	events := make(chan *appv1.ApplicationWatchEvent, watchAPIBufferSize)
	// Mimic watch API behavior: send ADDED events if no resource version provided
	// If watch API is executed for one application when emit event even if resource version is provided
	// This is required since single app watch API is used for during operations like app syncing and it is
	// critical to never miss events.
	if q.GetResourceVersion() == "" || q.GetName() != "" {
        // 리소스 버전이 없었거나 이름이 없었으면 
		apps, err := s.appLister.List(selector)
		if err != nil {
			return fmt.Errorf("error listing apps with selector: %w", err)
		}
		sort.Slice(apps, func(i, j int) bool {
			return apps[i].QualifiedName() < apps[j].QualifiedName()
		})
		for i := range apps {
			sendIfPermitted(*apps[i], watch.Added)
            // 앱의 이벤트 타입을 생성 (ADDED)로 추가 해서 보냄
		}
	}
	// 이벤트 채널을 구독할 수 있도록 만들고
	unsubscribe := s.appBroadcaster.Subscribe(events) 
	defer unsubscribe() //함수 종료되면 구독 해제 되게 해둠
	for {
		select {
		case event := <-events:
			sendIfPermitted(event.Application, event.Type)
		case <-ws.Context().Done():
			return nil
		}
	}
}
```

<참고용>
```golang
//server/application/broadcaster.go
type subscriber struct {
	ch      chan *appv1.ApplicationWatchEvent
	filters []func(*appv1.ApplicationWatchEvent) bool
}

// ...

// Subscribe forward application informer watch events to the provided channel.
// The watch events are dropped if no receives are reading events from the channel so the channel must have
// buffer if dropping events is not acceptable.
func (b *broadcasterHandler) Subscribe(ch chan *appv1.ApplicationWatchEvent, filters ...func(event *appv1.ApplicationWatchEvent) bool) func() {
	b.lock.Lock()
	defer b.lock.Unlock()
	subscriber := &subscriber{ch, filters} 
	b.subscribers = append(b.subscribers, subscriber) // 구독자를 추가  
	return func() { // 구독 해제하는 함수 반환
		b.lock.Lock()
		defer b.lock.Unlock()
		for i := range b.subscribers {
			if b.subscribers[i] == subscriber {
				b.subscribers = append(b.subscribers[:i], b.subscribers[i+1:]...)
				break
			}
		}
	}
}
```

ui에서 어떻게 argocd에 등록된 앱에 변경점을 감지하는지 정리해보면
- argocd ui에서는 resource version을 기반으로 변경을 감지
- resource version이 변경되었는지에 대해서는 서버로부터 이벤트 스트리밍을 통해 정보를 받음


---
```golang
// pkg/client/listers/application/v1alpha1/application.go
// List lists all Applications in the indexer.
func (s *applicationLister) List(selector labels.Selector) (ret []*v1alpha1.Application, err error) {
    // 캐시에서 리스트를 갖고오는 것 같음 
    // 여기서의 캐시는 client-go의 Indexer
    // https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md
	err = cache.ListAll(s.indexer, selector, func(m interface{}) {
		ret = append(ret, m.(*v1alpha1.Application))
	})
	return ret, err
}
```

그리고 applicationLister를 초기화 하는 흐름은
appcontroller에서 NewApplicationController를 초기화 하면서
newApplicationInformerAndLister를 호출하면서 
informer를 이용해 초기화 된다. 

```golang
// controller/appcontroller.go#L217
appInformer, appLister := ctrl.newApplicationInformerAndLister()

//...
// controller/appcontroller.go#L2271
lister := applisters.NewApplicationLister(informer.GetIndexer())
//
```

infomer는 NewShardIndexInformer로 초기화된다. 
https://pkg.go.dev/k8s.io/client-go/tools/cache#NewSharedIndexInformer


결국 이 흐름에서는 informer를 통해 앱의 정보를 로컬 캐시에 업데이트 받고
lister는 캐시에서 가져와서 앱의 정보를 클라이언트에 전달하는 방식이다.


