# Java 8 `groupingBy()`를 잘 쓰자

## 문제

>[ { yyyymmddhh: count }, { yyyymmddhh: count }, ... ] 와 같이 연월일시로 되어 있는 데이터를
>
>[ { yyyymmdd: countByDate }, { yyyymmdd: countByDate }, ... ] 와 같이 연월일로 집계하자.

### 데이터

```java
    class DateAndCount {
        private String yyyymmdd;
        private Long count;

        public DateAndCount(String yyyymmdd, Long count) {
            this.yyyymmdd = yyyymmdd;
            this.count = count;
        }
    }

    private List<DateAndCount> dateAndCounts =
            Arrays.asList(
                    new DateAndCount("2018080101", 1L)
                    , new DateAndCount("2018080102", 3L)
                    , new DateAndCount("2018080103", 5L)
                    , new DateAndCount("2018080104", 7L)
                    , new DateAndCount("2018080105", 9L)
                    , new DateAndCount("2018080201", 2L)
                    , new DateAndCount("2018080202", 4L)
                    , new DateAndCount("2018080203", 6L)
                    , new DateAndCount("2018080204", 8L)
                    , new DateAndCount("2018080205", 10L)
            );
```


## 어설픈 `Stream`과 `groupingBy()` 사용

자주 안 쓰다보니 `groupingBy()` 사용 방식을 까먹은데다가, 퇴근 전이다보니 다음과 같이 대충 작성해서 푸쉬하고 일단 퇴근했다. ㅋㅋ

```java
    @Test
    public void whenStreamingWithUglyGroupingBy__thenUglyCode() {
        final List<DateAndCount> results = this.dateAndCounts.stream()
                .collect(groupingBy(dnc -> dnc.yyyymmdd.substring(0, 8)))
                .entrySet().stream()
                .sorted(Map.Entry.<String, List<DateAndCount>>comparingByKey())
                .map(entry -> {
                    long sumByDate = entry.getValue().stream()
                            .map(dnc -> dnc.count)
                            .reduce((a, b) -> a + b)
                            .orElse(0L);
                    return new DateAndCount(entry.getKey(), sumByDate);
                })
                .collect(toList());

        assertThat(results.size()).isEqualTo(2);
        assertThat(results.get(0).yyyymmdd).isEqualTo("20180801");
        assertThat(results.get(0).count).isEqualTo(25L);
        assertThat(results.get(1).yyyymmdd).isEqualTo("20180802");
        assertThat(results.get(1).count).isEqualTo(30L);
    }
```

정렬도 지저분, 집계도 지저분, 여러모로 지저분하다. 지저분한거 알지만 일단 퇴근 셔틀은 타야 되니까..=3=3

## 정갈한 `Stream`과 `groupingBy()` 사용

집에 와서 API 문서를 보고 가다듬었다.

```java
    @Test
    public void whenStreamingWithGoodGroupingBy__thenSimpleCode() {
        final List<DateAndCount> results = this.dateAndCounts.stream()
                .collect(groupingBy(dnc -> dnc.yyyymmdd.substring(0, 8), TreeMap::new, summingLong(dnc -> dnc.count)))
                .entrySet().stream()
                .map(entry -> new DateAndCount(entry.getKey(), entry.getValue()))
                .collect(toList());

        assertThat(results.size()).isEqualTo(2);
        assertThat(results.get(0).yyyymmdd).isEqualTo("20180801");
        assertThat(results.get(0).count).isEqualTo(25L);
        assertThat(results.get(1).yyyymmdd).isEqualTo("20180802");
        assertThat(results.get(1).count).isEqualTo(30L);
    }
```
`groupingBy()`는 `분류 기준(Function<? super T,? extends K> classifier)` 뿐 아니라, `결과 Map 구성 방식(Supplier<M> mapFactory)`, `집계 방식(Collector<? super T,A,D> downstream)` 이렇게 3가지를 인자로 받을 수 있다. 

이를 잘 활용하면 위와 같이 정렬과 집계를 한 방에 해결할 수 있다.

### 나중에 추가

`groupingBy()`나 `summingLong()`에는 null 방어 코드가 필요하다.

예를 들어 `yyyymmdd`가 null 일 때는 `99991231`로 모으고, `count`가 null 일 때는 0 으로 처리한다면, 
``` java
groupingBy(dnc -> Optional.ofNullable(dnc.yyyymmdd).orElse("99991231").substring(0, 8)), TreeMap::new, summingLong(dnc -> Optional.ofNullable(dnc.count).orElse(0L))
```
와 같이 해주면 된다.


## 정갈한 `for`

하는 김에 가장 친숙한 for문도 해봤다. 의외로 제일 깔끔한 것 같다..

```java
    @Test
    public void whenForLoop__thenSimpleCode() {
        final Map<String, Long> dateAndCountByDateMap = new TreeMap<>();
        for (DateAndCount dnc: this.dateAndCounts) {
            dateAndCountByDateMap.merge(dnc.yyyymmdd.substring(0, 8), dnc.count, (a, b) -> a + b);
        }
        final List<DateAndCount> results = new ArrayList<>();
        for (Map.Entry<String, Long> entry: dateAndCountByDateMap.entrySet()) {
            results.add(new DateAndCount(entry.getKey(), entry.getValue()));
        }

        assertThat(results.size()).isEqualTo(2);
        assertThat(results.get(0).yyyymmdd).isEqualTo("20180801");
        assertThat(results.get(0).count).isEqualTo(25L);
        assertThat(results.get(1).yyyymmdd).isEqualTo("20180802");
        assertThat(results.get(1).count).isEqualTo(30L);
    }
```

역시 `for`가 짱이다..=3=3

---
## groupingBy()의 코드 절감 효과

아래의 두 코드는 정확히 같은 일을 한다.

```java
List<DataStoreStatistics> dataStoreStatsList = this.dataStoreStatisticsRepository.findByDate(searchStartDate, searchEndDate);

// for 버전
Map<Long, List<DataStoreStatistics>> datasByDataSourceIdMap = new HashMap<>();
for (DataStoreStatistics dss: dataStoreStatsList) {
    Long dataSourceId = dss.getDataSourceId();
    List<DataStoreStatistics> dssListByDataSource = 
        Optional.ofNullable(datasByDataSourceIdMap.get(dataSourceId)).orElse(new ArrayList<>());
    dssListByDataSource.add(dss);
    datasByDataSourceIdMap.put(dataSourceId, dssListByDataSource);
}

// groupingBy 버전
Map<Long, List<DataStoreStatistics>> datasByDataSourceIdMap = dataStoreStatsList.stream()
        .collect(groupingBy(DataStoreStatistics::getDataSourceId));
```
