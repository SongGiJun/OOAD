# Coding Agent System 다이어그램

제공된 클래스 다이어그램(CLIController, AgentExecutor, LLMClient, Tool, SecurityManager, ResultCollector, FileTool, CommandTool, Sandbox)에서 도출한 설계안입니다.

## 1. 작업 실행 시퀀스

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant CLI as CLIController
    participant Agent as AgentExecutor
    participant LLM as LLMClient
    participant File as FileTool
    participant Security as SecurityManager
    participant Command as CommandTool
    participant Sandbox as Sandbox
    participant Collector as ResultCollector

    User->>CLI: 작업 프롬프트 입력
    CLI->>Agent: run(prompt)
    activate Agent
    Agent->>Agent: buildContext()
    Agent->>LLM: requestPlan(context)
    LLM-->>Agent: ExecutionPlan
    Agent->>LLM: requestChanges(plan)
    LLM-->>Agent: ChangeSet

    loop 최초 실행 + 실패 시 수정 재시도
        Agent->>Agent: applyChanges(changes)
        loop 각 파일 변경 ToolCall
            Agent->>File: execute(call)
            File->>Security: check(call)
            Security-->>File: PermissionDecision
            alt 권한 허용
                File->>File: perform(call)
                File-->>Agent: ToolResult
                Agent->>Collector: record(result)
            else 권한 거부
                File-->>Agent: ToolResult(denied)
                Agent->>Collector: recordError(message)
                Note over Agent,Collector: 정책 위반은 작업 실패로 처리
            end
        end

        opt 변경 적용 성공 시
            Agent->>Agent: runBuildAndTest()
            loop 빌드 및 테스트 명령
                Agent->>Command: execute(call)
                Command->>Security: check(call)
                Security-->>Command: PermissionDecision
                alt 권한 허용
                    Command->>Sandbox: execute(command, path)
                    Sandbox-->>Command: CommandResult
                    Command-->>Agent: ToolResult
                    Agent->>Collector: record(result)
                else 권한 거부
                    Command-->>Agent: ToolResult(denied)
                    Agent->>Collector: recordError(message)
                    Note over Agent,Collector: 정책 위반은 작업 실패로 처리
                end
            end
            Agent->>Collector: recordChecks(checkResult)
            alt 빌드 및 테스트 통과
                Note over Agent,Collector: status=Success, 반복 종료
            else 실패 및 retryCount < maxRetries
                Agent->>Agent: retryWithFix(error)
                Agent->>LLM: requestFix(error)
                LLM-->>Agent: ChangeSet
            else 재시도 소진
                Agent->>Collector: recordError(error)
                Note over Agent,Collector: status=Failed, 반복 종료
            end
        end
    end

    Agent->>Collector: collect(status)
    Collector-->>Agent: TaskResult
    Agent-->>CLI: TaskResult
    deactivate Agent
    CLI->>CLI: renderOutput(result)
    CLI-->>User: 작업 결과 표시
```

## 2. AgentExecutor 상태 머신

```mermaid
stateDiagram-v2
    direction TB
    state "대기" as Idle
    state "컨텍스트 구성" as Context
    state "계획 생성 요청" as Planning
    state "변경안 생성 요청" as RequestChanges
    state "파일 변경 적용" as Applying
    state "빌드 및 테스트" as Validating
    state "오류 수정 요청" as Fixing
    state "중단 요청 처리" as Stopping
    state "결과 수집" as Collecting
    state "완료" as Completed
    state "실패" as Failed
    state "중단됨" as Stopped

    [*] --> Idle
    Idle --> Context: run(prompt) / 초기화
    Context --> Planning: buildContext() 성공
    Planning --> RequestChanges: requestPlan() 성공
    RequestChanges --> Applying: requestChanges() 성공
    Applying --> Validating: applyChanges() 완료
    Validating --> Collecting: 검증 통과 / 성공 상태
    Validating --> Fixing: 검증 실패 [retryCount < maxRetries]
    Fixing --> Applying: requestFix() 성공 / retryCount++
    Validating --> Collecting: 검증 실패 [retryCount >= maxRetries] / 실패 상태

    Context --> Collecting: 컨텍스트 구성 실패
    Planning --> Collecting: 계획 요청 실패
    RequestChanges --> Collecting: 변경안 요청 실패
    Applying --> Collecting: 정책 거부 또는 변경 적용 실패
    Fixing --> Collecting: 수정안 요청 실패

    Context --> Stopping: stop()
    Planning --> Stopping: stop()
    RequestChanges --> Stopping: stop()
    Applying --> Stopping: stop()
    Validating --> Stopping: stop()
    Fixing --> Stopping: stop()
    Stopping --> Collecting: 중단 확인 / 중단 상태

    Collecting --> Completed: collect(status) [성공]
    Collecting --> Failed: collect(status) [실패]
    Collecting --> Stopped: collect(status) [중단]
    Completed --> [*]
    Failed --> [*]
    Stopped --> [*]

    note right of Stopping
      stopRequested를 설정하고
      안전한 중단 지점에서 종료한다.
      실행 중인 Sandbox 즉시 종료는
      별도의 취소 전파 경로가 필요하다.
    end note
```

## 모델 해석 및 보완점

- FileTool.execute / CommandTool.execute는 추상 클래스 Tool.execute를 상속해서 호출하는 것으로 모델링했습니다. perform()은 구체 도구의 구현입니다.
- `TaskStatus`의 실제 enum 항목은 원본 그림에 없어 성공/실패/중단을 개념적 상태로 사용했습니다.
- 시퀀스의 반복 구간은 정상 실행과 자동 수정 재시도를 요약한 것입니다. 권한 거부 시에는 빌드/테스트로 진행하지 않고 실패 결과를 수집하도록 구현해야 합니다.
- 현재 클래스 다이어그램에는 AgentExecutor.stop()에서 실행 중인 CommandTool / Sandbox.terminate()로 중단을 전달하는 명시적 API가 없습니다. 즉시 명령 취소가 필요하면 CommandTool 또는 Tool의 cancel/stop 인터페이스를 설계해야 합니다.
- 중단은 LLM 호출 중에도 요청될 수 있으므로 `stopRequested`의 동시성 안전성과 취소 확인 지점을 구현에서 정의해야 합니다.
