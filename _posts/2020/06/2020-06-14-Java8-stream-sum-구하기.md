---
title: "Java8 stream sum 구하기"
last_modified_at: 2020-06-14T16:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/06/2020-06-14-title.jpg
  og_image: /assets/images/post/2020/06/2020-06-14-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Luis Quintero on Unsplash"
  
tags:
  - Java
category: #카테고리
  - Java
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



Java8 숫자 타입 List에서 합계를 구하는 방법입니다.  Integer, Long, Double은 Stream의 reduce()나 전용 Stream을 이용해서 바로 sum 구할 수 있고 BigInteger, BigDecimal은 reduce를 이용해서 구합니다.

## Integer

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);  

// Stream의 reduce 이용  
Integer sum1 = numbers.stream().reduce(0, Integer::sum);  

// IntStream의 sum 이용  
int sum2 = numbers.stream().mapToInt(i -> i).sum();
```

## Long

```java
List<Long> numbers = Arrays.asList(1L, 2L, 3L, 4L, 5L);

// Stream의 reduce 이용
Long sum1 = numbers.stream().reduce(0L, Long::sum);

// LongStream의 sum 이용
long sum2 = numbers.stream().mapToLong(i -> i).sum();
```

## Double

```java
List<Double> numbers = Arrays.asList(1.1D, 2.2D, 3.3D, 4.4D, 5.5D);

// Stream의 reduce 이용
Double su1 = numbers.stream().reduce(0D, Double::sum);

// DoubleStream의 sum 이용
double sum2 = numbers.stream().mapToDouble(i -> i).sum();
```

## BigInteger

```java
List<BigInteger> numbers = Arrays.asList(
        BigInteger.valueOf(1),
        BigInteger.valueOf(2),
        BigInteger.valueOf(3),
        BigInteger.valueOf(4),
        BigInteger.valueOf(5));

// Stream의 reduce 이용  
BigInteger sum = numbers.stream().reduce(BigInteger.ZERO, BigInteger::add);
```

## BigDecimal

```java
List<BigDecimal> numbers = Arrays.asList(
        BigDecimal.valueOf(1.1),
        BigDecimal.valueOf(2.2),
        BigDecimal.valueOf(3.3),
        BigDecimal.valueOf(4.4),
        BigDecimal.valueOf(5.5));

// Stream의 reduce 이용  
BigDecimal sum = numbers.stream().reduce(BigDecimal.ZERO, BigDecimal::add);
```

끝.