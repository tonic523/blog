---
title: "Spring Boot에서 redis로 순위 기능 구현하기"
date: 2023-09-21 08:00:00
update: 2023-09-21
tags:
- redis
- NoSQL
- RedisTemplate
---

# 개요

사용자 별로 특정 지표를 기준으로 순위를 매겨 노출시키는 요구사항을 받았습니다. 순위에 대한 기능을 redis로 어떻게 구현했는지 그리고 redis를 사용한 이유에 대해 기술했습니다.

# 요구사항

요구사항은 간단한 예시를 기반으로 설명하겠습니다.

> 학교 홈페이지에 로그인 했을 때 학생별로 이번달 수학 점수 평균이 몇위인지 보여주세요.
> 
- 학생들은 매일 아침 수학 시험을 풀고 점수가 나옵니다.
- 이번달 학생별로 수학 점수의 평균값을 기반으로 순위를 정합니다.
    - 1일 - 50점, 2일 60점, 3일 70점 → 평균 점수: 60점

### 테이블 구조

- Student
    - id
    - name
- Math
    - id
    - student_id
    - score
    - created_at

## RDBMS에서 조회하여 구현하기

각 학생별로 이번달 수학 점수의 평균값에 대한 순위를 구할 때는 다음과 같은 로직이 필요합니다.

1. 이번달 수학 시험을 푼 모든 학생들의 수학 점수 평균값을 구합니다.
2. 수학 점수 평균값을 기준으로 순위를 매깁니다.
3. 로그인한 학생은 몇위인지 구합니다.

```java
public class MathRankService {

    private final MathRepository mathRepository;

    public int getCurrentMonthRanking(Long studentId) {
        // 현재 월의 시작일과 종료일을 계산
        LocalDate currentDate = LocalDate.now();
        LocalDate startDate = currentDate.withDayOfMonth(1);
        LocalDate endDate = currentDate.withDayOfMonth(currentDate.lengthOfMonth());
    
        // 1. 이번달 수학 점수 리스트를 가져옴
        List<Math> mathScores = mathRepository.findByCreatedAtBetween(startDate, endDate);
    
        // 2. 모든 학생의 수학 점수를 그룹화하여 학생별 평균을 계산
        Map<Long, Double> studentAverages = mathScores.stream()
                .collect(Collectors.groupingBy(
                        math -> math.getStudent().getId(),
                        Collectors.averagingDouble(Math::getScore)));
    
        // 3. 로그인한 학생이 몇위인지 구한다.
        Double currentStudentAverage = studentAverages.get(studentId);
        long ranking = studentAverages.values().stream()
                .filter(average -> average > currentStudentAverage)
                .count() + 1; // 1을 더해 순위를 1부터 시작하도록 함
    
        return (int) ranking;
    }
}
```

여기서 고려해야 하는 부분은 수학 점수 리스트를 가져올 때 입니다.

- `List<Math> mathScores = mathRepository.findByCreatedAtBetween(startDate, endDate);`

이번달 학생별로 풀었던 모든 수학 점수들을 조회하는 쿼리입니다.

> 8월 1일 ~ 20일 까지 2000명의 학생들이 매일 수학 점수를 풀었을 경우
> 

math 테이블에는 `20 * 2000 = 40000개의 데이터`가 쌓이게 됩니다. 그럼 위 메서드를 호출할 때 마다 40000개의 데이터를 조회한 후 학생별로 평균 점수를 계산하여 그룹화한 다음 순위를 노출시키고 있습니다. 메서드의 시간복잡도는 `O(N)` 입니다. 해당 테이블에서 필터링을 하더라도 요구사항의 특성 상 학생의 수 또는 시험의 빈도에 따라 많은 데이터를 불러올 수 밖에 없습니다.

문제는 만약 학생들이 로그인했을 때 무조건 순위를 보여줘야 한다는 요구사항이라면, 2000명의 학생들이 1번씩 로그인만 한다고 가정했을 때 해당 메서드는 2000번 호출될 정도로 자주 호출되는 API 연결될 수 있다는 것 입니다.

## Redis 적용하기

`Redis`의 `Sorted Set`을 활용하여 구현해 보겠습니다. 간단하게 redis란 키-값 형태의 데이터를 저장하고 관리하는 NoSQL 데이터베이스로, 주로 캐싱, 세션 관리, 실시간 분석 등 다양한 용도로 사용됩니다. 인메모리 데이터베이스로 데이터를 메모리에 저장하여 빠른 읽기 및 쓰기 작업을 지원합니다.

### Sorted Set

Redis 데이터 구조 중 하나로, 정렬된 데이터를 저장하고 관리하는데 사용합니다. 각 데이터 멤버(membeR)에 점수(score)를 부여하고, 멤버를 점수에 따라 정렬하는 자료구조입니다. 다음은 명령어 별 특징과 시간 복잡도입니다.

1. **ZADD**: 하나 또는 여러 개의 멤버를 Sorted Set에 추가합니다.
    - 시간 복잡도: O(log(N)) + O(M*log(N)), 여기서 M은 추가된 멤버의 수입니다.
2. **ZRANK**: 주어진 멤버가 오름차순으로 정렬된 Sorted Set에서 몇 번째로 작은 멤버인지 조회합니다.
    - 시간 복잡도: O(log(N))
3. **ZREVRANK**: 주어진 멤버가 내림차순으로 정렬된 Sorted Set에서 몇 번째로 큰 멤버인지 조회합니다.
    - 시간 복잡도: O(log(N))

다른 명령어들도 많지만, 우선 제가 사용한 예시에서 사용되는 명령어를 기준으로 작성했습니다. 실제 구현은 다음과 같이 했습니다.

1. 학생들이 시험을 풀고 점수가 나올 때 마다 redis의 Sorted Set에 업데이트한다.
2. 로그인을 했을 때 Sorted Set에서 학생 식별자를 통해 순위를 구한다.

**Sorted Set 구조**
- key(자료구조를 식별하는 값): MonthlyMathRanking
- member
    - value: 학생 식별자(student_id)
    - score: 학생 별 수학 점수 평균값

```java
public class MathRankService {

    private static final String REDIS_KEY = "MonthlyMathRanking";

    private final MathRepository mathRepository;
    private final RedisTemplate redisTemplate;

    // 학생들이 로그인할 때 실행
    public int getCurrentMonthRanking(Long studentId) {
        return redisTemplate.opsForZSet().reverseRank(REDIS_KEY, accountId) + 1; // redis ZREVRANK 명령어 수행
    }

    // 학생들이 시험을 풀 때 마다 실행
    public void updateMonthlyMathScore(Long studentId) {
		    List<Math> mathScores = mathRepository.findAllByStudentIdByCreatedAtBetween(studentId, startDate, endDate);
        int averageScore = mathScores.stream()
            .mapToInt(Math::getScore)
            .average();
        redisTemplate.opsForZSet().add(REDIS_KEY, studentId, averageScore); // redis ZADD 명령어 수행
    }
}
```

Spring Boot 기반의 프로젝트에서는 RedisTemplate을 통해 Redis와 통신할 수 있습니다.

ZREVRANK 명령어를 사용하는 `redisTemplate.opsForZSet().reverseRank(REDIS_KEY, accountId)` 는 시간복잡도를 봤을 때 `O(logN)`입니다. 따라서 기존에 table의 데이터 기준 `O(N)`으로 조회했던 RDBMS 방식과 비교했을 때 성능적으로 상당한 이점을 가져갈 수 있습니다.

ZADD 명령어를 사용하는 `redisTemplate.opsForZSet().add(REDIS_KEY, studentId, averageScore)` 의 시간 복잡도는 `O(logN) + O(log(N))` 입니다. 기존 RDBMS에 추가했을 때 시간복잡도(O(1))에 비해서는 성능이 떨어질 수 있지만, 순위를 조회할 때의 성능 차이와 비교했을 때 미미한 정도라고 볼 수 있습니다. 그리고 순위를 조회할 때 즉 학생들이 로그인하는 횟수와 학생들이 시험을 보는 횟수를 비교했을 때 조회가 훨씬 많이 발생하기 때문에 조회의 성능을 살리는 것이 좋다고 생각합니다.

> redis를 사용했을 때 고려할 부분 또는 단점은 없을까요?

고려할 부분은 어떤 데이터를 기준으로 언제 업데이트 시킬지가 중요한 것 같습니다. 앞선 예시에서는 학생들이 시험을 볼 때 마다 즉 데이터베이스의 데이터가 업데이트 될 때 마다 redis도 업데이트 되도록 설계되어 있습니다. 이 과정에서 에러 또는 데이터 유실이 발생할 경우 데이터베이스와 redis에 담긴 정보의 정합성이 깨질 수 있어 이를 보완할 수 있는 로직이 필요할수 있습니다.

redis라는 메모리형 DB의 단점이 있습니다. 성능을 위해 메모리에 정보를 저장하는 구조이기 때문에 redis 서버에 문제가 발생할 경우 메모리에 있는 데이터가 유실될 가능성이 있습니다. 필요에 따라 redis 자체에서도 백업용 파일을 만드는 설정을 하여 지속적인 백업과 복구가 필요합니다.

### 참고

[Redis data structures - SET](https://www.joinc.co.kr/w/man/12/REDIS/RedisWithJoinc/part04)