argocd 프로젝트의 기본 구조를 분석해보자.

---

# main 구조

먼저 argocd 프로젝트의 출발지는 cmd/ main.go이다. main함수의 구조는 다음과 같다.

```go
package main

// ...

// https://github.com/argoproj/argo-cd/blob/4f6e4088efc789a8cb44d3e25a444467c46d761f/cmd/main.go#L27
func main() {
	var command *cobra.Command

	// ✅ 현재 실행중인 파일 경로를 인자로 전달: 빌드된 파일 이름이 binary name으로 들어감(이거 그럼 디버깅 어떻게하지)
	binaryName := filepath.Base(os.Args[0])
	if val := os.Getenv(binaryNameEnv); val != "" {
		binaryName = val
	}

	isCLI := false
	// binary 이름에 따라 분기
	switch binaryName {
	// ✅ argocd cli인 경우
	case "argocd", "argocd-linux-amd64", "argocd-darwin-amd64", "argocd-windows-amd64.exe":
		command = cli.NewCommand()
		isCLI = true
	// ✅ argocd server인 경우
	case "argocd-server":
		command = apiserver.NewCommand()
	case "argocd-application-controller":
		command = appcontroller.NewCommand()
	// ...
	}
	util.SetAutoMaxProcs(isCLI) // gmp p setting

	// command 실행
	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}

```

위의 구조는 되게 심플하게 돌아간다. 실행한 binary이름을 찾고 binary에 맞는 command를 끼고 command를 실행한다. 이 과정에서 cobra라는 라이브러리를 사용한다.

---

# cobra

예를 들어 아래와 같은 코드를 짠 경우,

```go
package main

import (
	"fmt"

	"github.com/spf13/cobra"
)

var name string

var rootCmd = &cobra.Command{
	Use:   "app",                      // 명령어 이름
	Short: "App is a simple CLI tool", // 간단한 설명
	Long: `App is a CLI tool built with Cobra.
This is an example application to demonstrate how Cobra works.`, // 상세 설명
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Hello, Cobra!")
	},
}

var greetCmd = &cobra.Command{
	Use:   "greet",
	Short: "Prints a greeting message",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Printf("Hello, %s!\n", name)
	},
}

func init() {
	greetCmd.Flags().StringVarP(&name, "name", "n", "World", "Name to greet") // 플래그 추가
	rootCmd.AddCommand(greetCmd)
}

func main() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
	}
}

```

이렇게 실행하면

```go
 ./main greet --help
```

이렇게 리턴한다.

```go
Prints a greeting message

Usage:
  app greet [flags]

Flags:
  -h, --help          help for greet
  -n, --name string   Name to greet (default "World")
flangdu@DESKTOP-SPRNMEM:/mnt/c/Users/dx/work-root/pr
```

그냥 cli도구. 중요한 것은 cobra에서 Run 메서드와 cmd를 execute하는 부분을 찾는 것이다.

---

# cli app 생성 동작

cli 코드는 가장 먼저 root.go에서 확인할 수 있다.

```go
// https://github.com/argoproj/argo-cd/blob/master/cmd/argocd/commands/root.go
func NewCommand() *cobra.Command {
	// ...
	command.AddCommand(initialize.InitCommand(NewApplicationCommand(&clientOpts)))
	// ...
	return command
}
```

여기서 함수를 하나씩 읽어보면 딱 봐도 application 관련 커맨드를 등록하는 함수가 보인다. `command.AddCommand(initialize.InitCommand(NewApplicationCommand(&clientOpts)))` 이 시그니처인데, 내부를 들어가보자.

```go
// https://github.com/argoproj/argo-cd/blob/96d0226a4963d9639aea81ec1d3a310fed390133/cmd/argocd/commands/app.go#L65
func NewApplicationCommand(clientOpts *argocdclient.ClientOptions) *cobra.Command {
	command := &cobra.Command{ ... }
	// ...
	command.AddCommand(NewApplicationCreateCommand(clientOpts))
	// ...
	return command
}

```

바로 밑에 함수 정의가 있는데 해당 부분을 확인해보자. 주요 메서드는 결국 Run이다.

```go
// https://github.com/argoproj/argo-cd/blob/96d0226a4963d9639aea81ec1d3a310fed390133/cmd/argocd/commands/app.go#L118
func NewApplicationCreateCommand(clientOpts *argocdclient.ClientOptions) *cobra.Command {
	// 클로저에서 사용할 변수
	var (
		appOpts      cmdutil.AppOptions
		fileURL      string
		appName      string
		upsert       bool
		labels       []string
		annotations  []string
		setFinalizer bool
		appNamespace string
	)
	// ...
	command := &cobra.Command {
		Use: "app", // 명령어 이름 argocd app ~~ 
		// ...
		Run: func(c *cobra.Command, args []string) {
			
		},
	}
	// 사용할 변수 플래깅	
	command.Flags().StringVar(&appName, "name", "", "A name for the app, ignored if a file is set (DEPRECATED)")
	command.Flags().BoolVar(&upsert, "upsert", false, "Allows to override application with the same name even if supplied application spec is different from existing spec")
	command.Flags().StringVarP(&fileURL, "file", "f", "", "Filename or URL to Kubernetes manifests for the app")
	command.Flags().StringArrayVarP(&labels, "label", "l", []string{}, "Labels to apply to the app")
	command.Flags().StringArrayVarP(&annotations, "annotations", "", []string{}, "Set metadata annotations (e.g. example=value)")
	command.Flags().BoolVar(&setFinalizer, "set-finalizer", false, "Sets deletion finalizer on the application, application resources will be cascaded on deletion")
}
```

command가 실행되면 `Run:`  의 필드로 받는 클로저가 실행된다.

```go
// https://github.com/argoproj/argo-cd/blob/96d0226a4963d9639aea81ec1d3a310fed390133/cmd/argocd/commands/app.go#L152
		Use:   "create APPNAME", // 명령어 이름
		// ...
		Run: func(c *cobra.Command, args []string) {
			// argocdClient 생성
			argocdClient := headless.NewClientOrDie(clientOpts, c)
		
			// ...
			// apps을 파일로부터 가져옴: 일단 어디선가 apps을 만들어서 가져옴
			apps, err := cmdutil.ConstructApps(fileURL, appName, labels, annotations, args, appOpts, c.Flags())
			errors.CheckError(err)

			// app을 순회하면서
			for _, app := range apps {
				// ...
				if appNamespace != "" {
					app.Namespace = appNamespace
				}
				if setFinalizer {
					app.Finalizers = append(app.Finalizers, "resources-finalizer.argocd.argoproj.io")
				}
				
				// argoClient에서 applicationClient를 만듦
				// grpc 서버 예상
				// ✅ 여기 함수 시그니처가 특이한데, io.Closer와 applicationClient를 같이 리턴
				conn, appIf := argocdClient.NewApplicationClientOrDie()
				defer argoio.Close(conn)
				
				// 생성 요청 생성
				appCreateRequest := application.ApplicationCreateRequest{
					Application: app,
					Upsert:      &upsert, // upsert 인자 전달
					Validate:    &appOpts.Validate,
				}

				// app 존재 여부 확인
				// Get app before creating to see if it is being updated or no change
				existing, err := appIf.Get(ctx, &application.ApplicationQuery{Name: &app.Name})
				unwrappedError := grpc.UnwrapGRPCStatus(err).Code()
				// As part of the fix for CVE-2022-41354, the API will return Permission Denied when an app does not exist.
				if unwrappedError != codes.NotFound && unwrappedError != codes.PermissionDenied {
					errors.CheckError(err)
				}
			
				// ✅ 생성 전달
				created, err := appIf.Create(ctx, &appCreateRequest)
				errors.CheckError(err)

				// 액션에 따라 application 동작 리턴
				var action string
				if existing == nil {
					action = "created"
				} else if !hasAppChanged(existing, created, upsert) {
					action = "unchanged"
				} else {
					action = "updated"
				}

				fmt.Printf("application '%s' %s\n", created.ObjectMeta.Name, action)
			}
		},

```

그러니까 해당 함수 동작을 요약하면 application 생성 템플릿을 가져와서 appClient를 만들고 생성 요청을 어딘가 전달한다.

먼저 `argocdClient.NewApplicationClientOrDie()` 내부를 보자. 정의로 이동하면 해당 함수를 확인할 수 있다.

```go
// https://github.com/argoproj/argo-cd/blob/96d0226a4963d9639aea81ec1d3a310fed390133/pkg/apiclient/apiclient.go#L686C1-L692C2
func (c *client) NewApplicationClientOrDie() (io.Closer, applicationpkg.ApplicationServiceClient) {
	conn, appIf, err := c.NewApplicationClient()
	if err != nil {
		log.Fatalf("Failed to establish connection to %s: %v", c.ServerAddr, err)
	}
	return conn, appIf
}

// https://github.com/argoproj/argo-cd/blob/96d0226a4963d9639aea81ec1d3a310fed390133/pkg/apiclient/apiclient.go#L668C1-L675C2
func (c *client) NewApplicationClient() (io.Closer, applicationpkg.ApplicationServiceClient, error) {
	conn, closer, err := c.newConn()
	if err != nil {
		return nil, nil, err
	}
	appIf := applicationpkg.NewApplicationServiceClient(conn)
	return closer, appIf, nil
}

// https://github.com/argoproj/argo-cd/blob/96d0226a4963d9639aea81ec1d3a310fed390133/pkg/apiclient/apiclient.go#L489C1-L489C66
func (c *client) newConn() (*grpc.ClientConn, io.Closer, error) {
	// ...
}
```

역시 파고 들어가면 grpc 클라이언트 커넥션을 만드는 부분이 나온다. 이 부분은 깊게 확인하지 않는다. grpc client 연결에 대한 부분은 따로 다루지 않으려고 한다.

참고로 application에 대한 .proto 형식은 여기서 확인 가능하다.

https://github.com/argoproj/argo-cd/blob/master/server/application/application.proto

---

# api server 라이프사이클

cli로 받은 요청은 argocd api server가 처리한다. 따라서 server에 대한 코드를 분석해야 한다. 그럼 다시 cli 실행 부분부터 api server 동작까지 흘러가는 방식으로 진행해보자.

```go
	// ...		
	case "argocd-server":
		command = apiserver.NewCommand()
	// ...
```

`apiserver.NewCommand()` 내부로 들어간다. 내부 구조는 심플하다. cliName에 대응하는 이벤트를 바로 설정하므로 해당 이벤트가 호출된다.

```go
// https://github.com/argoproj/argo-cd/blob/4f6e4088efc789a8cb44d3e25a444467c46d761f/cmd/argocd-server/commands/argocd_server.go#L55
func NewCommand() *cobra.Command {
	// 함수 내부 변수 세팅
	// ...
	command := &cobra.Command{
		Use:               cliName,
		DisableAutoGenTag: true,
		Run: func(c *cobra.Command, args []string) {
			// some logic
			// ...
		}
		// 커맨드 변수 flagging
		// ...
		
	// cache 설정
	tlsConfigCustomizerSrc = tls.AddTLSFlagsToCmd(command)
	cacheSrc = servercache.AddCacheFlagsToCmd(command, cacheutil.Options{
		OnClientCreated: func(client *redis.Client) {
			redisClient = client
		},
	})
	repoServerCacheSrc = reposervercache.AddCacheFlagsToCmd(command, cacheutil.Options{FlagPrefix: "repo-server-"})
	return command
```

중요한 건 Run 내부이므로 해당 함수를 분석해야 한다. **argocd는 서버로 동작하니 이 부분에서 blocking call이 있어야 한다**는 합리적인 의심을 해볼 수 있다.

다음은 Run의 내부이다.

```go
// https://github.com/argoproj/argo-cd/blob/4f6e4088efc789a8cb44d3e25a444467c46d761f/cmd/argocd-server/commands/argocd_server.go#L105
		Run: func(c *cobra.Command, args []string) {
			
			// argocd server 설정
			// ...
			argocd := server.NewServer(ctx, argoCDOpts, appsetOpts) // argocd 생성
			argocd.Init(ctx) // informer 고루틴 실행
			for {
				var closer func()
				serverCtx, cancel := context.WithCancel(ctx)
				lns, err := argocd.Listen() // tcp 서버 열기
				errors.CheckError(err)
				// ...
				argocd.Run(serverCtx, lns) // 서버 실행
				if closer != nil {
					closer()
				}
				cancel()
				if argocd.TerminateRequested() {
					break
				}
			}
		},
```

보통 일반적인 서버 블로킹은 Run같은 메서드가 한다. 위에 딱 의심스러운 argocd.Run이 있다. 이 부분을 더 들어가보자. Run 메서드가 해야 할 작업은 명확하다. 여러 대의 서버 및 mux를 고루틴으로 열고 해당 고루틴의 종료를 감지 & waitgroup 대기하는 것이다.

```go
// https://github.com/argoproj/argo-cd/blob/dfbfdbab1188dfb26b454e47ac06c70ed484c066/server/server.go#L535
func (a *ArgoCDServer) Run(ctx context.Context, listeners *Listeners) {
	// ...
	// http / https mux & grpc server setting
	svcSet := newArgoCDServiceSet(a)
	a.serviceSet = svcSet
	grpcS, appResourceTreeFn := a.newGRPCServer()
	grpcWebS := grpcweb.WrapServer(grpcS)
	var httpS *http.Server
	var httpsS *http.Server
	if a.useTLS() {
		httpS = newRedirectServer(a.ListenPort, a.RootPath)
		httpsS = a.newHTTPServer(ctx, a.ListenPort, grpcWebS, appResourceTreeFn, listeners.GatewayConn, metricsServ)
	} else {
		httpS = a.newHTTPServer(ctx, a.ListenPort, grpcWebS, appResourceTreeFn, listeners.GatewayConn, metricsServ)
	}
	// ...

	// goroutine으로 서버 호스팅 및 에러 감지
	go func() { a.checkServeErr("grpcS", grpcS.Serve(grpcL)) }()
	go func() { a.checkServeErr("httpS", httpS.Serve(httpL)) }()
	if a.useTLS() {
		go func() { a.checkServeErr("httpsS", httpsS.Serve(httpsL)) }()
		go func() { a.checkServeErr("tlsm", tlsm.Serve()) }()
	}
	go a.watchSettings()
	go a.rbacPolicyLoader(ctx)
	go func() { a.checkServeErr("tcpm", tcpm.Serve()) }()
	go func() { a.checkServeErr("metrics", metricsServ.Serve(listeners.Metrics)) }()
	if !cache.WaitForCacheSync(ctx.Done(), a.projInformer.HasSynced, a.appInformer.HasSynced) {
		log.Fatal("Timed out waiting for project cache to sync")
	}

	// 종료 시점 컬백함수 호출
	shutdownFunc := func() {
		// ...
		var wg gosync.WaitGroup

		// Shutdown http server
		wg.Add(1)
		go func() {
			defer wg.Done()
			err := httpS.Shutdown(shutdownCtx)
			// ...
		}()

		if a.useTLS() {
			// Shutdown https server
			wg.Add(1)
			go func() {
				defer wg.Done()
				err := httpsS.Shutdown(shutdownCtx)
				// ...
			}()
		}

		// Shutdown gRPC server
		wg.Add(1)
		go func() {
			defer wg.Done()
			grpcS.GracefulStop()
		}()

		// Shutdown metrics server
		wg.Add(1)
		go func() {
			defer wg.Done()
			err := metricsServ.Shutdown(shutdownCtx)
			// ...
		}()

		if a.useTLS() {
			// Shutdown tls server
			wg.Add(1)
			go func() {
				defer wg.Done()
				tlsm.Close()
			}()
		}

		// Shutdown tcp server
		wg.Add(1)
		go func() {
			defer wg.Done()
			tcpm.Close()
		}()

		c := make(chan struct{})
		// This goroutine will wait for all servers to conclude the shutdown
		// process
		go func() { // 모든 서버 종료 이벤트 수신 시 채널 닫기
			defer close(c)
			wg.Wait()
		}()

		// 채널 닫힐 때까지 대기
		select {
		case <-c:
			log.Info("All servers were gracefully shutdown. Exiting...")
		// ...
		}
	}
	
	// 시그널 등록
	a.shutdown = shutdownFunc
	signal.Notify(a.stopCh, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)
	a.available.Store(true)

	// 종료 대기
	select {
	case signal := <-a.stopCh:
		log.Infof("API Server received signal: %s", signal.String())
		// SIGUSR1 is used for triggering a server restart
		if signal != syscall.SIGUSR1 {
			a.terminateRequested.Store(true)
		}
		a.shutdown()
	case <-ctx.Done():
		log.Infof("API Server: %s", ctx.Err())
		a.terminateRequested.Store(true)
		a.shutdown()
	}
}

```

서버 실행을 다시 한 번 살펴보자. 서버에 대한 라이프사이클은 다음 로직을 따른다.

1. cli실행
2. 실행하면 server 생성 → tcp 서버 열기 → 서버 실행 과정으로 동작
3. 서버 실행
4. 종료 call이 떨어지면 waitgroup을 통해 올바르게 종료하는지 확인
5. 모든 서버가 종료되기를 대기하고 채널 수신 후 프로세스 종료

---

# grpc 요청 동작

그렇지만 서버 라이프사이클은 크게 중요하지 않다. 중요한 건 어떻게 요청이 처리되는가이다. 이를 확인하려면 grpcServer에 어떻게 구현한 rpc가 등록되는지 찾아야 한다.

다시 Run 메서드를 확인해보자.

```go
// https://github.com/argoproj/argo-cd/blob/dfbfdbab1188dfb26b454e47ac06c70ed484c066/server/server.go#L535
func (a *ArgoCDServer) Run(ctx context.Context, listeners *Listeners) {
	// ...
	// http / https mux & grpc server setting
	grpcS, appResourceTreeFn := a.newGRPCServer()
	// ...
```

여기서 `a.newGRPCServer()` 가 호출된다. 이 내부를 들어가자. 내부에는 grpc 서버가 생성되고 다양한 서비스가 서버에 등록되는 것을 확인할 수 있다.

```go
// https://github.com/argoproj/argo-cd/blob/dfbfdbab1188dfb26b454e47ac06c70ed484c066/server/server.go#L876C1-L876C87
func (a *ArgoCDServer) newGRPCServer() (*grpc.Server, application.AppResourceTreeFn) {
	// ...
	// grpc 서버 생성
	grpcS := grpc.NewServer(sOpts...)
	
	// grpc 서버에 service 레지스터
	versionpkg.RegisterVersionServiceServer(grpcS, a.serviceSet.VersionService)
	clusterpkg.RegisterClusterServiceServer(grpcS, a.serviceSet.ClusterService)
	applicationpkg.RegisterApplicationServiceServer(grpcS, a.serviceSet.ApplicationService) // application service 등록은 여기에 존재
	applicationsetpkg.RegisterApplicationSetServiceServer(grpcS, a.serviceSet.ApplicationSetService)
	notificationpkg.RegisterNotificationServiceServer(grpcS, a.serviceSet.NotificationService)
	repositorypkg.RegisterRepositoryServiceServer(grpcS, a.serviceSet.RepoService)
	repocredspkg.RegisterRepoCredsServiceServer(grpcS, a.serviceSet.RepoCredsService)
	sessionpkg.RegisterSessionServiceServer(grpcS, a.serviceSet.SessionService)
	settingspkg.RegisterSettingsServiceServer(grpcS, a.serviceSet.SettingsService)
	projectpkg.RegisterProjectServiceServer(grpcS, a.serviceSet.ProjectService)
	accountpkg.RegisterAccountServiceServer(grpcS, a.serviceSet.AccountService)
	certificatepkg.RegisterCertificateServiceServer(grpcS, a.serviceSet.CertificateService)
	gpgkeypkg.RegisterGPGKeyServiceServer(grpcS, a.serviceSet.GpgkeyService)
	// Register reflection service on gRPC server.
	reflection.Register(grpcS)
	grpc_prometheus.Register(grpcS)
	errorsutil.CheckError(a.serviceSet.ProjectService.NormalizeProjs())
	return grpcS, a.serviceSet.AppResourceTreeFn
}

```

관심있는 app에 대한 서비스는 `applicationpkg.RegisterApplicationServiceServer(grpcS, a.serviceSet.ApplicationService)` 이 부분에서 등록되는 것으로 보인다. 그럼 s.serviceSet.ApplicationService가 어디서 초기화되는지 찾아보자. 이 부분은 다시 Run 서버의 다음 부분이다.

```go
// https://github.com/argoproj/argo-cd/blob/dfbfdbab1188dfb26b454e47ac06c70ed484c066/server/server.go#L535
func (a *ArgoCDServer) Run(ctx context.Context, listeners *Listeners) {
	// ...
	// http / https mux & grpc server setting
	svcSet := newArgoCDServiceSet(a)
	a.serviceSet = svcSet
```

이 함수를 확인하자. 일단 application 관련 정보만 찾아보자. 온 종류의 서비스를 다 만드는데 지금은 확인할 필요가 없다. 결국 app용으로 호출되는 건 `application.NewServer(...)` 이다.

```go
// https://github.com/argoproj/argo-cd/blob/dfbfdbab1188dfb26b454e47ac06c70ed484c066/server/server.go#L979C1-L1067C2
func newArgoCDServiceSet(a *ArgoCDServer) *ArgoCDServiceSet {
	kubectl := kubeutil.NewKubectl()
	// ...
	applicationService, appResourceTreeFn := application.NewServer(
		a.Namespace,
		a.KubeClientset,
		a.AppClientset,
		a.appLister,
		a.appInformer,
		nil,
		a.RepoClientset,
		a.Cache,
		kubectl,
		a.db,
		a.enf,
		projectLock,
		a.settingsMgr,
		a.projInformer,
		a.ApplicationNamespaces,
		a.EnableK8sEvent,
	)
	// ...
	return &ArgoCDServiceSet{
		ClusterService:        clusterService,
		RepoService:           repoService,
		RepoCredsService:      repoCredsService,
		SessionService:        sessionService,
		ApplicationService:    applicationService,
		AppResourceTreeFn:     appResourceTreeFn,
		ApplicationSetService: applicationSetService,
		ProjectService:        projectService,
		SettingsService:       settingsService,
		AccountService:        accountService,
		NotificationService:   notificationService,
		CertificateService:    certificateService,
		GpgkeyService:         gpgkeyService,
		VersionService:        versionService,
	}
}
```

여기서 application.NewServer의 시그니처는 이렇다.

```go
// https://github.com/argoproj/argo-cd/blob/dfbfdbab1188dfb26b454e47ac06c70ed484c066/server/application/application.go#L99C1-L116C60
func NewServer(
	// ...
) (application.ApplicationServiceServer, AppResourceTreeFn) {
	s := &Server{
		// ...
	}
	return s, s.getAppResources
}
```

이 함수를 살펴보면 리턴으로 &Server를 리턴한다. 이 함수는 다음의 메서드를 갖는다. 이 메서드가 grpc 서비스를 구현하는 메서드인 것을 확인할 수 있다. 5개만 보자.

```go
func (s *Server) Create(ctx context.Context, q *application.ApplicationCreateRequest) (*appv1.Application, error)
func (s *Server) Delete(ctx context.Context, q *application.ApplicationDeleteRequest) (*application.ApplicationResponse, error)
func (s *Server) DeleteResource(ctx context.Context, q *application.ApplicationResourceDeleteRequest) (*application.ApplicationResponse, error)
func (s *Server) Get(ctx context.Context, q *application.ApplicationQuery) (*appv1.Application, error)
func (s *Server) GetApplicationSyncWindows(ctx context.Context, q *application.ApplicationSyncWindowsQuery) (*application.ApplicationSyncWindowsResponse, error)
// ...
```

여기서 Create 요청이 처리된다. 이 함수는 한 번 따라서 읽어보자. 상당히 직관적으로 작성되어 있다.

```go
// https://github.com/argoproj/argo-cd/blob/dfbfdbab1188dfb26b454e47ac06c70ed484c066/server/application/application.go#L314C1-L314C116
func (s *Server) Create(ctx context.Context, q *application.ApplicationCreateRequest) (*appv1.Application, error) {
	// Application nil 체크
	if q.GetApplication() == nil {
		return nil, fmt.Errorf("error creating application: application is nil in request")
	}
	a := q.GetApplication()

	// rbac check
	if err := s.enf.EnforceErr(ctx.Value("claims"), rbacpolicy.ResourceApplications, rbacpolicy.ActionCreate, a.RBACName(s.ns)); err != nil {
		return nil, err
	}

	// get project lock
	s.projectLock.RLock(a.Spec.GetProject())
	defer s.projectLock.RUnlock(a.Spec.GetProject())

	validate := true
	if q.Validate != nil {
		validate = *q.Validate
	}

	// get app project
	proj, err := s.getAppProject(ctx, a, log.WithField("application", a.Name))
	if err != nil {
		return nil, err
	}

	// validate normalize app
	err = s.validateAndNormalizeApp(ctx, a, proj, validate)
	if err != nil {
		return nil, fmt.Errorf("error while validating and normalizing app: %w", err)
	}

	appNs := s.appNamespaceOrDefault(a.Namespace)

	// 가능한 ns 확인
	if !s.isNamespaceEnabled(appNs) {
		return nil, security.NamespaceNotPermittedError(appNs)
	}

	// Don't let the app creator set the operation explicitly. Those requests should always go through the Sync API.
	if a.Operation != nil {
		log.WithFields(log.Fields{
			"application":            a.Name,
			argocommon.SecurityField: argocommon.SecurityLow,
		}).Warn("User attempted to set operation on application creation. This could have allowed them to bypass branch protection rules by setting manifests directly. Ignoring the set operation.")
		a.Operation = nil
	}

	// k8s object app 생성
	created, err := s.appclientset.ArgoprojV1alpha1().Applications(appNs).Create(ctx, a, metav1.CreateOptions{})
	// 생성 성공인 경우
	if err == nil {
		s.logAppEvent(created, ctx, argo.EventReasonResourceCreated, "created application")
		s.waitSync(created)
		return created, nil
	}
	
	// err가 존재하는 경우 아래 로직: 에러 이유가 이미 존재하는 앱이 아닌 경우
	if !apierr.IsAlreadyExists(err) {
		return nil, fmt.Errorf("error creating application: %w", err)
	}

	// 기존 app 가져와서 비교 로직 진행 및 upsert
	// act idempotent if existing spec matches new spec
	existing, err := s.appLister.Applications(appNs).Get(a.Name)
	if err != nil {
		return nil, status.Errorf(codes.Internal, "unable to check existing application details (%s): %v", appNs, err)
	}
	equalSpecs := reflect.DeepEqual(existing.Spec, a.Spec) &&
		reflect.DeepEqual(existing.Labels, a.Labels) &&
		reflect.DeepEqual(existing.Annotations, a.Annotations) &&
		reflect.DeepEqual(existing.Finalizers, a.Finalizers)

	if equalSpecs {
		return existing, nil
	}
	if q.Upsert == nil || !*q.Upsert {
		return nil, status.Errorf(codes.InvalidArgument, "existing application spec is different, use upsert flag to force update")
	}
	if err := s.enf.EnforceErr(ctx.Value("claims"), rbacpolicy.ResourceApplications, rbacpolicy.ActionUpdate, a.RBACName(s.ns)); err != nil {
		return nil, err
	}
	updated, err := s.updateApp(existing, a, ctx, true)
	if err != nil {
		return nil, fmt.Errorf("error updating application: %w", err)
	}
	return updated, nil
}
```

동작을 요약하면 다음과 같다.

1. 프로젝트 정보 조회
2. 네임스페이스 체크
3. application 생성
4. upsert 처리

그럼 이제 궁금해진다. app 생성 후 sync와 같은 작업이 되는 경우 어떻게 argocd는 이를 처리하는 것인가? 다음은 application controller를 살펴보자.