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