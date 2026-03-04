# 시스템 아키텍처 다이어그램

[← 메인으로 돌아가기](../README.md)

---

## 📋 다이어그램 사용 가이드

| 다이어그램 | 권장 사용 상황 |
|-----------|--------------|
| 전체 시스템 아키텍처 | README 첫인상, 초기 설명 |
| Command/Event 처리 흐름 | 면접·프레젠테이션에서 핵심 설계 설명 |
| 장애 복구 플로우 | 운영 관점 질문 대응 |
| 장애 영향도 맵 | "안정성을 어떻게 보장하나요?" 대응 |
| 데이터 흐름 | 기술 문서·아키텍처 리뷰 |
| 배포 아키텍처 | "확장성은?" 질문 대응 |
| 5대 설계 원칙 연결도 | "왜 이렇게 설계했나요?" 대응 |

---

## 📋 목차

1. [전체 시스템 아키텍처](#전체-시스템-아키텍처)
2. [5대 설계 원칙 연결도](#5대-설계-원칙-연결도)
3. [Command/Event 처리 흐름](#commandevent-처리-흐름)
4. [장애 복구 플로우](#장애-복구-플로우)
5. [장애 영향도 맵](#장애-영향도-맵)
6. [데이터 흐름](#데이터-흐름)
7. [배포 아키텍처 (Zone 기반 확장)](#배포-아키텍처-zone-기반-확장)
8. [Supporting Portfolios 연결도](#supporting-portfolios-연결도)

---

## 전체 시스템 아키텍처

```mermaid
graph TB
    subgraph Client["🎮 Client Layer"]
        UC["Unity Client\nServer-authoritative"]
    end

    subgraph GameServer["⚡ Game Server Layer (C#)"]
        GS["C# Game Server\nTCP/IP"]
        GM["GameLoop Tick\n50ms"]
        MS["Memory State\nPlayer/World"]
        CMD["Command Handler\nValidation"]
        EPB["Event Publisher\nKafka Producer"]
    end

    subgraph EventStream["📨 Event Stream Layer"]
        KF["Kafka\nEvent Stream"]
        T1["game.events.player"]
        T2["game.events.combat"]
        T3["game.snapshot"]
    end

    subgraph PlatformServer["🌐 Platform Server Layer (TypeScript/Bun.js)"]
        PS["Platform Server"]
        EH["Event Handler\nConsumer"]
        API["REST API\nElysiaJS"]
    end

    subgraph StorageLayer["💾 Storage Layer"]
        RD["Redis\nHot Snapshot\nTTL: 5min"]
        MG["MongoDB\nCold Snapshot\nPermanent"]
        MY["MySQL\nPersistent Data\nACID"]
    end

    UC -->|"TCP Request"| GS
    GS -->|"Response"| UC
    GS --> GM --> MS
    GS --> CMD --> EPB
    EPB -->|"Fire-and-Forget"| KF
    KF --> T1 & T2 & T3
    T1 & T2 & T3 --> EH --> PS --> API
    GS -.->|"Save"| RD
    GS -.->|"Save"| MG
    PS -->|"Persist"| MY

    style GM fill:#e74c3c,color:#fff
    style KF fill:#f39c12,color:#fff
    style RD fill:#c0392b,color:#fff
    style MG fill:#27ae60,color:#fff
```

---

## 5대 설계 원칙 연결도

```mermaid
graph TD
    subgraph Principles["5대 설계 원칙"]
        P1["① 실시간 판정은\n메모리에서 즉시 확정"]
        P2["② 기록과 확장은\n이벤트로 분리"]
        P3["③ 장애는 전파되지 않고\n국소화"]
        P4["④ 기능 추가가\n기존 흐름을 깨지 않음"]
        P5["⑤ 운영자가\n시스템을 이해할 수 있음"]
    end

    subgraph Implementation["구현체"]
        GS["GameLoop\nIn-Memory State"]
        KAFKA["Kafka\nFire-and-Forget"]
        ISO["장애 영향도\n매트릭스"]
        OCP["Kafka Topic\n구독 확장"]
        OPS["Admin Dashboard\n운영 가이드"]
    end

    P1 --> GS
    P2 --> KAFKA
    P3 --> ISO
    P4 --> OCP
    P5 --> OPS

    style P1 fill:#2980b9,color:#fff
    style P2 fill:#2980b9,color:#fff
    style P3 fill:#2980b9,color:#fff
    style P4 fill:#2980b9,color:#fff
    style P5 fill:#2980b9,color:#fff
```

---

## Command/Event 처리 흐름

### 패킷부터 DB까지의 완전한 여정

```mermaid
sequenceDiagram
    participant C as Unity Client
    participant GS as Game Server
    participant M as Memory (World)
    participant K as Kafka
    participant PS as Platform Server
    participant DB as Database

    Note over C,DB: 1. 클라이언트 요청
    C->>GS: MoveRequest(newPosition)

    Note over GS: 2. 검증 & 판정
    GS->>GS: Validate Move
    GS->>GS: Check Distance
    GS->>GS: Check Cooldown

    Note over GS,M: 3. 상태 변경 (메모리 — 즉시 확정)
    GS->>M: player.Position = newPosition
    M-->>GS: State Updated

    Note over GS,C: 4. 즉시 응답
    GS->>C: MoveResponse(confirmedPosition)

    Note over GS,K: 5. Domain Event 발행 (비동기)
    GS->>K: PlayerMovedEvent
    Note over K: Fire-and-Forget<br/>게임 서버는 대기하지 않음

    Note over K,PS: 6. Event 소비
    K->>PS: PlayerMovedEvent
    PS->>PS: Idempotency Check

    Note over PS,DB: 7. DB 영속화
    PS->>DB: INSERT player_movements
    DB-->>PS: Success

    Note over C,DB: ✅ 전체 흐름 완료 — DB 저장 실패가 게임플레이를 막지 않음
```

### 핵심 타이밍

```mermaid
gantt
    title 처리 타이밍 분석
    dateFormat X
    axisFormat %Lms

    section Client
    요청 전송    :0, 5
    응답 대기    :5, 50
    화면 갱신    :50, 60

    section Game Server
    패킷 수신    :5, 10
    검증        :10, 15
    상태 변경    :15, 20
    응답 전송    :20, 50
    Event 발행  :20, 25

    section Kafka
    Event 저장  :25, 100

    section Platform
    Event 소비  :100, 150
    DB 저장     :150, 250
```

---

## 장애 복구 플로우

```mermaid
flowchart TD
    START(["게임 서버 크래시"]) --> DETECT["Health Check 실패 감지"]
    DETECT --> NEW["새 서버 인스턴스 시작"]

    NEW --> CHECK_R{"Redis Snapshot\n존재?"}

    CHECK_R -->|"Yes"| LOAD_R["Redis 스냅샷 로드"]
    LOAD_R --> RESTORE_R["플레이어 상태 복원"]
    RESTORE_R --> FAST(["✅ 복구 완료\nRTO: 10초"])

    CHECK_R -->|"No"| CHECK_M{"MongoDB Snapshot\n존재?"}

    CHECK_M -->|"Yes"| LOAD_M["MongoDB 스냅샷 로드"]
    LOAD_M --> RESTORE_M["플레이어 상태 복원"]
    RESTORE_M --> SLOW(["⚠️ 복구 완료\nRTO: 2~3분"])

    CHECK_M -->|"No"| INIT["초기 상태로 시작"]
    INIT --> REPLAY["Kafka Event Replay"]
    REPLAY --> MANUAL(["🔧 수동 복구\nRTO: 5~10분"])

    style START fill:#e74c3c,color:#fff
    style FAST fill:#27ae60,color:#fff
    style SLOW fill:#f39c12,color:#fff
    style MANUAL fill:#7f8c8d,color:#fff
```

---

## 장애 영향도 맵

```mermaid
graph LR
    subgraph Failures["장애 발생"]
        GS_F["게임 서버 DOWN"]
        RD_F["Redis DOWN"]
        KF_F["Kafka DOWN"]
        MY_F["MySQL DOWN"]
        PS_F["플랫폼 서버 DOWN"]
    end

    subgraph GameEffect["게임플레이 영향"]
        GS_G["🔴 중단"]
        RD_G["🟡 순간 지연"]
        KF_G["🟢 정상"]
        MY_G["🟢 정상"]
        PS_G["🟢 정상"]
    end

    subgraph RecordEffect["기록 영향"]
        GS_R["🟡 일시 중단"]
        RD_R["🟢 정상"]
        KF_R["🟡 일시 중단"]
        MY_R["🟡 일시 중단"]
        PS_R["🟡 일시 중단"]
    end

    GS_F --> GS_G --> GS_R
    RD_F --> RD_G --> RD_R
    KF_F --> KF_G --> KF_R
    MY_F --> MY_G --> MY_R
    PS_F --> PS_G --> PS_R
```

---

## 데이터 흐름

### 실시간 경로 vs 영속 경로

```mermaid
flowchart TB
    subgraph Hot["⚡ 실시간 경로 (Hot Path)"]
        C["Client Request"]
        GS["Game Server\nMemory"]
        RED["Redis\nHot Cache"]
        C --> GS --> RED
    end

    subgraph Cold["❄️ 영속 경로 (Cold Path)"]
        KF["Kafka\nEvent Stream"]
        PS["Platform Server"]
        MG["MongoDB\nEvent Log"]
        MY["MySQL\nPersistent"]
        KF --> PS --> MG
        PS --> MY
    end

    GS -->|"Fire-and-Forget"| KF
    GS -.->|"Periodic Snapshot"| MG

    style Hot fill:#fff3cd
    style Cold fill:#d4edda
```

---

## 배포 아키텍처 (Zone 기반 확장)

```mermaid
graph TB
    subgraph Internet["Internet"]
        CLI["Unity Clients"]
    end

    subgraph Gateway["Gateway Layer"]
        LB["Load Balancer"]
    end

    subgraph ZoneLayer["Zone Layer"]
        ZC["Zone Coordinator"]
        Z1["Zone Server 1\n100 CCU"]
        Z2["Zone Server 2\n100 CCU"]
        ZN["Zone Server N\n100 CCU"]
        ZC --> Z1 & Z2 & ZN
    end

    subgraph EventLayer["Event Layer"]
        KAFKA["Kafka Cluster"]
    end

    subgraph PlatformLayer["Platform Layer"]
        PS1["Platform #1"]
        PS2["Platform #2"]
    end

    subgraph DataLayer["Data Layer"]
        REDIS["Redis Cluster"]
        MONGO["MongoDB RS"]
        MYSQL["MySQL MS"]
    end

    CLI --> LB --> ZC
    Z1 & Z2 & ZN -->|"Events"| KAFKA
    KAFKA --> PS1 & PS2
    PS1 & PS2 --> REDIS & MONGO & MYSQL

    style ZC fill:#2980b9,color:#fff
    style KAFKA fill:#f39c12,color:#fff
```

---

## Supporting Portfolios 연결도

```mermaid
graph LR
    VAMPIRE["🎮 Vampire Survival\n클라이언트 권한 한계 체감"] -->|"Server-authoritative 필요성"| MAIN
    SHADER["🎨 Shader Experiments\nGPU 프레임 비용 측정"] -->|"20fps 동기화 설계 근거"| MAIN
    REACT["💻 React Experiments\n상태 관리 · Snapshot 검증"] -->|"Admin Dashboard 기반"| MAIN
    IOT["🌡️ Production IoT\nKafka · Adapter Pattern"] -->|"프로덕션 설계 판단 증명"| MAIN
    NGINX["🛡️ Nginx Gateway\nTLS · Rate Limit · Docker"] -->|"운영 인프라 설계"| MAIN
    COIN["📊 Coin Data API\nDI · 외부 격리 · 정규화"] -->|"도메인 일반화 증명"| MAIN

    MAIN["🚩 Main Portfolio\nServer-authoritative\nEvent-driven Platform"]

    style MAIN fill:#2c3e50,color:#fff
    style VAMPIRE fill:#8e44ad,color:#fff
    style IOT fill:#1a6b3c,color:#fff
```

---

[← 메인으로 돌아가기](../README.md)

**Last Updated**: 2026-03-04
