프롬프트: 높이 N을 입력받아 중앙 정렬된 피라미드 형태의 문자 패턴을 출력할건데, 홀수번째 행은 *, 짝수 번 째 행은 #으로 출력할거야. c언어로 실행할거고, visual studio에서 실행할거야
```c

#include <stdio.h>

int main(void)
{
    int N;
    int i, j;

    printf("높이를 입력하세요: ");
    scanf_s("%d", &N);

    for (i = 1; i <= N; i++)
    {
        // 왼쪽 공백 출력
        for (j = 1; j <= N - i; j++)
        {
            printf(" ");
        }

        // 홀수 번째 행은 *, 짝수 번째 행은 #
        for (j = 1; j <= 2 * i - 1; j++)
        {
            if (i % 2 == 1)
            {
                printf("*");
            }
            else
            {
                printf("#");
            }
        }

        // 줄바꿈
        printf("\n");
    }

    return 0;
}

```
<img width="319" height="170" alt="image" src="https://github.com/user-attachments/assets/b7493f2f-496a-480c-93ee-0b48ce3f0a5a" />
