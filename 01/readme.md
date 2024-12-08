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