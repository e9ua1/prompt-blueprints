# Spring Batch Deep Dive 레포지토리 제작 프롬프트

나는 "Spring Batch Deep Dive" 레포지토리를 만들려고 해.
대용량 데이터를 Chunk 단위로 처리하는 원리, Job/Step/Tasklet이 어떻게 실행되는지, 실패 시 Retry/Skip 전략을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "수백만 건 데이터를 안정적으로 처리하는 메커니즘"

**핵심 차별화**:
1. Chunk-Oriented Processing 원리 완전 분해
2. Job Repository와 메타데이터 관리
3. Retry/Skip/Restart 실패 복구 전략
4. Partitioning으로 대용량 병렬 처리

**타겟 독자**:
- 배치 작업을 해야 하는데 Spring Batch를 모르는 개발자
- Chunk 처리가 무엇인지 모르는 개발자
- 배치 실패 시 재시작 전략을 모르는 개발자
- 대용량 데이터를 병렬로 처리해야 하는 개발자

**선행 학습**:
- Spring Core Deep Dive (Bean, DI 이해)
- Spring Data & Transaction (Transaction 관리 필수)

---

## 🎯 1단계: 전체 구조 설계

다음 6개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Batch Architecture (6개 문서)
- Job, Step, Tasklet 관계와 구조
- JobLauncher와 Job 실행 메커니즘
- Job Repository (메타데이터 저장)
- Job Instance vs Job Execution vs Step Execution
- Job Parameters와 실행 컨텍스트
- Batch Configuration (@EnableBatchProcessing)

### Chapter 2: Chunk-Oriented Processing (7개 문서)
- Chunk 처리 원리 (Read → Process → Write)
- ItemReader 종류와 구현 (JpaPagingItemReader, JdbcCursorItemReader)
- ItemProcessor 체인과 조건부 처리
- ItemWriter 구현 (JpaItemWriter, JdbcBatchItemWriter)
- Chunk Size 튜닝 (10 vs 100 vs 1000)
- Cursor vs Paging 선택 기준
- Custom ItemReader/Writer 작성

### Chapter 3: Job Execution & Flow Control (6개 문서)
- Sequential Flow (Step1 → Step2 → Step3)
- Conditional Flow (on().to().from())
- Decision (JobExecutionDecider)
- Parallel Steps (Split)
- Flow Externalization
- Job Listener (BeforeJob, AfterJob)

### Chapter 4: Error Handling & Recovery (7개 문서)
- Skip 전략 (SkipPolicy, SkipLimit)
- Retry 전략 (RetryPolicy, RetryTemplate)
- Skip vs Retry 선택 기준
- JobRestartException과 재시작
- ExecutionContext를 통한 상태 저장
- Fault Tolerance 설정
- Custom Skip/Retry Policy

### Chapter 5: Partitioning & Parallel Processing (5개 문서)
- Partitioning 개념 (Manager Step, Worker Steps)
- Partitioner 구현 (데이터 분할)
- Grid Size와 Thread Pool
- Remote Partitioning (Master-Slave)
- Partitioning 성능 튜닝

### Chapter 6: Advanced Topics (4개 문서)
- Async ItemProcessor (비동기 처리)
- Multi-threaded Step
- JobScope vs StepScope
- Spring Batch Integration (메시지 기반 처리)

---

각 챕터는 **4~7개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 패턴이 필요한가
## 😱 흔한 실수 (Before - 비효율적인 배치)
## ✨ 올바른 패턴 (After - 최적화된 배치)
## 🔬 내부 동작 원리 (Spring Batch 소스 추적)
## 💻 실전 구현 (실행 가능한 Job)
## 📊 성능 비교 (처리 시간, 메모리 사용량)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 실제 배치 Job 코드 작성
2. Job 실행 로그 분석
3. Chunk Size에 따른 성능 측정
4. 실패 시나리오와 복구 전략
5. 대용량 데이터 처리 최적화

**실험 환경**:
- H2/MySQL 데이터베이스
- 100만 건 이상 테스트 데이터
- Job 실행 로그 분석
- JobRepository 메타데이터 확인

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**: 
   - Spring Data & Transaction README 참고
   - Spring Batch에 맞게 변형
   - 배치 처리 실무 강조
3. **챕터별 문서 작성**: 
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

<Spring Core Deep Dive README를 여기에 붙여넣기>

<Spring Data & Transaction README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조로 Spring Batch Deep Dive 버전을 만들어줘.

**차이점**:
- 주제: 대용량 배치 처리
- 초점: Chunk 처리 + 실패 복구
- 실험: 성능 측정, 메타데이터 분석

---

## 💡 핵심 분석 대상

```java
// Chunk-Oriented Step 실행 흐름
while (ItemReader.read() != null) {
    // 1. Chunk Size만큼 Read
    for (int i = 0; i < chunkSize; i++) {
        Object item = itemReader.read();
        if (item == null) break;
        
        // 2. Process
        Object processed = itemProcessor.process(item);
        chunk.add(processed);
    }
    
    // 3. Write (Chunk 단위)
    itemWriter.write(chunk.getItems());
    
    // 4. Transaction Commit
}

// Job 실행
JobLauncher.run(job, jobParameters)
  → Job.execute()
    → Step.execute()
      → ChunkOrientedTasklet.execute()
        → doRead() → doProcess() → doWrite()
```

이 흐름을 완전히 분해!

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- 성능 테스트 시나리오 (100만 건, Chunk Size 비교 등)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
