---
title: "Java DecimalFormat Pattern"
last_modified_at: 2017-07-29T22:13:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2017/2017-07-29-numbers.jpg
  og_image: /assets/images/post/2017/2017-07-29-numbers.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Photo by Matt Hoffman on Unsplash]"
tags:
  - Java
  - DecimalFormat
category: #카테고리
  - Java
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


오랜만에 사용 하려면 기억나지 않는 `DecimalFormat` 클래스의 Number Format Pattern을 정리한다.

| **심볼** | **설명**                                                     |
| -------- | ------------------------------------------------------------ |
| 0        | 숫자 한자리를 표시한다. 숫자가 없는 경우 0을 표시한다.       |
| #        | 숫자 한자리를 표시한다. 숫자가 없는 경우 아무것도 표시하지 않는다. |
| .        | 소수점 표시                                                  |
| ,        | 그룹 표시                                                    |
| '        | 접두사, 접미사에 특정 문자를 표시 (화폐 표시)                |
| %        | 100을 곱해서 백분율로 표시                                   |

### 예제

```java
public class DecimalFormatTest {

    public static void main(String[] args) {
        print(18.714, "000.##");
        print(1234567, "#,###,###");
        print(0.57, "%");
        print(-3.456, "0.00");
        print(982.45, ".#####");
        print(17500, "'￦',###");
        print(5805.10, "'$',###.00");
    }

    private static void print(Number number, String pattern) {
        DecimalFormat decimalFormat = new DecimalFormat(pattern);
        String format = decimalFormat.format(number);
        System.out.println("Number: " + number + ", Pattern: " + pattern + ", result: " + format);
    }
}
```
### 실행결과
```
Number: 18.714, Pattern: 000.##, result: 018.71
Number: 1234567, Pattern: #,###,###, result: 1,234,567
Number: 0.57, Pattern: %, result: %57
Number: -3.456, Pattern: 0.00, result: -3.46
Number: 982.45, Pattern: .#####, result: 982.45
Number: 17500, Pattern: '￦',###, result: ￦17,500
Number: 5805.1, Pattern: '$',###.00, result: $5,805.10
```

끝.