#OOAD 4조

# Sequence Diagram

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

# State Diagram

```mermaid
stateDiagram-v2
    direction TB
    state "대기" as Idle
    state "실행 중" as Running
    state "중단 대기" as Stopping
    state "결과 수집" as Collecting
    state "완료" as Completed
    state "실패" as Failed
    state "중단됨" as Stopped

    [*] --> Idle
    Idle --> Running: run(prompt) / 실행 컨텍스트 초기화

    state Running {
        state "컨텍스트 구성" as Context
        state "계획 생성 요청" as Planning
        state "변경안 생성 요청" as RequestChanges
        state "파일 변경 적용" as Applying
        state "빌드 및 테스트" as Validating
        state "오류 수정 요청" as Fixing

        [*] --> Context
        Context --> Planning: 컨텍스트 구성 성공
        Planning --> RequestChanges: 계획 응답 수락
        RequestChanges --> Applying: 변경안 응답 수락
        Applying --> Validating: 변경 적용 완료
        Validating --> Fixing: 검증 실패 및 재시도 가능
        Fixing --> Applying: 수정안 응답 수락 / retryCount++
    }

    Running --> Stopping: stop() / 중단 요청 등록
    Stopping --> Collecting: 진행 중 작업이 안전 지점에 도달 / 중단 확정

    Validating --> Collecting: 검증 통과 / 성공 확정
    Validating --> Collecting: 검증 실패 및 재시도 소진 / 실패 확정
    Context --> Collecting: 컨텍스트 구성 실패
    Planning --> Collecting: 계획 요청 실패
    RequestChanges --> Collecting: 변경안 요청 실패
    Applying --> Collecting: 정책 거부 또는 변경 적용 실패
    Fixing --> Collecting: 수정안 요청 실패

    Collecting --> Completed: 성공
    Collecting --> Failed: 실패
    Collecting --> Stopped: 중단
    Completed --> [*]
    Failed --> [*]
    Stopped --> [*]
```
