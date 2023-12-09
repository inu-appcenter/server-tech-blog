# Spring Batch 모집글 상태 자동 변환

## 1. 문제 상황
- 스터디 게시글은 스터디 시작 날짜가 되면 자동으로 모집 마감 처리를 해야 한다

## 2. 해결 방안
- spring batch + spring scheduler 를 사용해서 해결해보자

## 3. 배치에 대해서
배치에서 Job이란 전체 배치 프로세스를 캡슐화한 도메인이면 Step의 순서를 정의하고 JobParameters를 받는다 그러면 JobInstance가 생기는데 JobInstance는 실질적으로 Job이 실행되는 객체이다 그리고 JobExecution이 실행된다

배치에서 Step이란 작업 처리의 단위이면 Chunk 기반 스탭, Tasklet 기반 스탭 두가지로 나뉜다

Chunk 기반 스탭은 대용량 데이터를 읽고 쓸때 사용되며 데이터를 읽고 쓰는거까지 하나의 트랜잭션에서 데이터를 처리한다
commitInterval만큼 데이터를 읽고 chunkSize만큼 데이터를 쓴다
![](https://velog.velcdn.com/images/wellbeing-dough/post/93132ec5-8fac-4000-a839-35678a14ccf6/image.png)

Tasklet 기반 스탭은 단순한 처리를 할 때 사용되며 데이터를 읽고 쓰는 과정을 하나로 퉁쳤다 이것 또한 하나의 트랜잭션에서 모든것을 처리한다

스터디 모집글 중 스터디 시작일이 오늘인 것과, 그중에서 스터디 마감 여부가 false인 데이터만 가져와서 스터디를 마감시키는거기 때문에 우리는 대용량 데이터도 아니며 간단한 작업이기 때문에 Tasklet을 사용하기로 했다

## 4. 구현

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class StudyPostJobConfig {

    public final JobBuilderFactory jobBuilderFactory;

    public final StepBuilderFactory stepBuilderFactory;

    private final StudyPostRepository studyPostRepository;

    @Bean("studyPostJob")
    public Job studyPostJob() {
        return jobBuilderFactory.get("studyPostJob")
                .incrementer(new RunIdIncrementer())
                .start(changeStudyPostCloseByDeadline())
                .on("FAILED")
                .stopAndRestart(changeStudyPostCloseByDeadline())
                .on("*")
                .end()
                .end()
                .build();
    }

    @JobScope
    @Bean("changeStudyPostCloseByDeadline")
    public Step changeStudyPostCloseByDeadline() {
        return stepBuilderFactory.get("changeStudyPostCloseByDeadline")
                .tasklet(studyPostTasklet())
                .build();
    }

    @StepScope
    @Bean("studyPostTasklet")
    public Tasklet studyPostTasklet() {
        return (contribution, chunkContext) -> {
            List<StudyPostEntity> studyPostList = studyPostRepository.findByStudyStartDate(LocalDate.now());
            	for(StudyPostEntity studyPost : studyPostList) {
  	               studyPost.closeStudyPost();
    	           studyPostRepository.save(studyPost);
            	}
            return RepeatStatus.FINISHED;
        };
    }

}
```


studyPostJob() 메서드:
- 메서드의 반환 타입은 Job
- 메서드 이름은 studyPostJob
- jobBuilderFactory를 사용하여 새로운 Job을 생성
- incrementer(new RunIdIncrementer()): Job의 실행을 식별하기 위한 RunId를 증가시키는 Incrementer를 설정 이는 Job 파라미터를 변경하여 새로운 Job Instance를 생성하는 데 사용
- .start(changeStudyPostCloseByDeadlineStep()): Job의 첫 번째 Step으로 changeStudyPostCloseByDeadlineStep() 메서드에서 반환된 Step을 설정
- .on("FAILED"): 만약 Step이 실패한 경우를 처리하는 옵션을 설정
- .stopAndRestart(changeStudyPostCloseByDeadlineStep()): Step이 실패한 경우 해당 Step을 중지하고 다시 시작하도록 설정
- .on("*"): 어떤 결과에 대해서도 처리하는 옵션을 설정
- .end(): Flow를 종료
- .end(): 더 이상의 Flow가 없으므로 Job을 종료
- .build(): 설정된 옵션을 기반으로 Job을 빌드하여 반환

changeStudyPostCloseByDeadlineStep() 메서드:
- 메서드의 반환 타입은 Step
- 메서드 이름은 changeStudyPostCloseByDeadlineStep
- @JobScope 어노테이션이 적용되어 Job 내에서만 사용 가능한 Step 빈으로 등록
- stepBuilderFactory를 사용하여 새로운 Step을 생성
- .tasklet(studyPostTasklet()): Step이 수행할 작업을 정의한 studyPostTasklet() 메서드에서 반환된 Tasklet을 설정
- .build(): 설정된 옵션을 기반으로 Step을 빌드하여 반환


studyPostTasklet() 메서드:
- 메서드의 반환 타입은 Tasklet
- 메서드 이름은 studyPostTasklet
- @StepScope 어노테이션이 적용되어 Step 내에서만 사용 가능한 Tasklet 빈으로 등록
- Tasklet은 Step에서 실행되는 실제 작업을 정의하는 함수형 인터페이스
- studyPostRepository.findByStudyStartDate(LocalDate.now()): 현재 날짜와 일치하는 StudyPostEntity를 조회
- studyPostRepository.save(studyPost): 조회된 StudyPostEntity의 상태를 변경하고 저장
- RepeatStatus.FINISHED: Step이 성공적으로 완료되었음을 나타내는 RepeatStatus를 반환

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class StudyPostScheduler {

   private final JobLauncher jobLauncher;
   private final Job job;

    @Scheduled(cron = "0 * * * * *")
    public void runJob() {

        try{
            jobLauncher.run(
                    job, new JobParametersBuilder().addString("dateTime", LocalDateTime.now().toString()).toJobParameters()
            );
        } catch (Exception e) {
            log.error(e.getMessage());
        }
    }
}

```

private final JobLauncher jobLauncher: 스프링 배치 Job을 실행하기 위한 JobLauncher를 주입받음
private final Job job: 실행할 스프링 배치 Job을 주입받음
JobParametersBuilder().addString("dateTime", LocalDateTime.now().toString()).toJobParameters(): 실행 시에 Job에 전달할 파라미터를 설정. "dateTime"이라는 키와 현재 시간을 문자열로 변환한 값을 파라미터로 설정 왜? 매 실행마다 JobParmeter가 변경되어야 해서

```java
public interface StudyPostRepository extends JpaRepository<StudyPostEntity, Long>, StudyPostRepositoryCustom{
    @Transactional
    @Modifying(clearAutomatically = true)
    @Query("UPDATE StudyPostEntity sp SET sp.close = true WHERE sp.studyStartDate = :studyStartDate AND sp.close = false")
    void closeStudyPostsByStartDate(@Param("studyStartDate") LocalDate studyStartDate);}

```

## 5. 개선
![](https://velog.velcdn.com/images/wellbeing-dough/post/40aae1c4-435a-4225-ae80-2b34b353e7c9/image.png)
저렇게 변경감지로 하나하나 업데이트 하니까 마감할 공고들이 10개면 update쿼리가 10개 나간다 직접 update 쿼리를 벌크연산으로 써주고 벌크연산은 영속성 컨텍스트에 반영되지 않기 때문에 혹시 모르니까 대응해주자(@Modifying(clearAutomatically = true))

```java
    @Transactional
    @Modifying(clearAutomatically = true)
    @Query("UPDATE StudyPostEntity sp SET sp.close = :close WHERE sp.studyStartDate = :studyStartDate AND sp.close = false")
    void closeStudyPostsByStartDate(@Param("studyStartDate") LocalDate studyStartDate, @Param("close") Boolean close);
}
```

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class StudyPostJobConfig {

    public final JobBuilderFactory jobBuilderFactory;

    public final StepBuilderFactory stepBuilderFactory;

    private final StudyPostRepository studyPostRepository;

    @Bean("studyPostJob")
    public Job studyPostJob() {
        return jobBuilderFactory.get("studyPostJob")
                .incrementer(new RunIdIncrementer())
                .start(changeStudyPostCloseByDeadline())
                .on("FAILED")
                .stopAndRestart(changeStudyPostCloseByDeadline())
                .on("*")
                .end()
                .end()
                .build();
    }

    @JobScope
    @Bean("changeStudyPostCloseByDeadline")
    public Step changeStudyPostCloseByDeadline() {
        return stepBuilderFactory.get("changeBoardStatus")
                .tasklet(studyPostTasklet())
                .build();
    }

    @StepScope
    @Bean("studyPostTasklet")
    public Tasklet studyPostTasklet() {
        return (contribution, chunkContext) -> {
            studyPostRepository.closeStudyPostsByStartDate(LocalDate.now(), true);
            return RepeatStatus.FINISHED;
        };
    }

}
```
코드가 훨씬 간결해졌다
![](https://velog.velcdn.com/images/wellbeing-dough/post/5807e485-63d3-4968-bc43-a57b34b84963/image.png)
성능도 훨씬 좋아졌다

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class StudyPostScheduler {

   private final JobLauncher jobLauncher;
   private final Job job;

    @Scheduled(cron = "0 0 4 * * *")
    public void runJob() {

        try{
            jobLauncher.run(
                    job, new JobParametersBuilder().addString("dateTime", LocalDateTime.now().toString()).toJobParameters()
            );
        } catch (Exception e) {
            log.error(e.getMessage());
        }
    }
}

```
그리고 스캐쥴러도 유저가 가장 없어서 서버 부하가 적은 시간(새벽 4시)에 적용 해 주었다