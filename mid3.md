프롬프트 : 숫자야구 게임을 만들거야. C언어로 visual studio에서 실행할거야

```c

#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main(void)
{
    int answer[3];
    int guess[3];
    int used[10] = { 0 };
    int strike, ball;
    int attempts = 0;

    srand((unsigned int)time(NULL));

    // 정답의 첫 번째 숫자 생성 (1~9)
    answer[0] = rand() % 9 + 1;
    used[answer[0]] = 1;

    // 나머지 두 숫자 생성
    for (int i = 1; i < 3; i++)
    {
        int digit;

        do
        {
            digit = rand() % 10;
        } while (used[digit]);

        answer[i] = digit;
        used[digit] = 1;
    }

    printf("===== 숫자야구 게임 =====\n");
    printf("서로 다른 세 자리 숫자를 맞혀 보세요!\n\n");

    while (1)
    {
        int number;

        printf("숫자 입력: ");

        if (scanf("%d", &number) != 1)
        {
            printf("잘못된 입력입니다. 숫자를 입력하세요.\n");

            while (getchar() != '\n');
            continue;
        }

        // 세 자리 숫자인지 확인
        if (number < 100 || number > 999)
        {
            printf("세 자리 숫자를 입력하세요.\n\n");
            continue;
        }

        guess[0] = number / 100;
        guess[1] = (number / 10) % 10;
        guess[2] = number % 10;

        // 중복 숫자 확인
        if (guess[0] == guess[1] ||
            guess[1] == guess[2] ||
            guess[0] == guess[2])
        {
            printf("중복되지 않는 숫자를 입력하세요.\n\n");
            continue;
        }

        attempts++;
        strike = 0;
        ball = 0;

        // 정답과 입력한 숫자 비교
        for (int i = 0; i < 3; i++)
        {
            for (int j = 0; j < 3; j++)
            {
                if (guess[i] == answer[j])
                {
                    if (i == j)
                    {
                        strike++;
                    }
                    else
                    {
                        ball++;
                    }
                }
            }
        }

        // 결과 출력
        if (strike == 3)
        {
            printf("3스트라이크! 정답입니다!\n");
            printf("총 시도 횟수: %d회\n", attempts);
            break;
        }
        else if (strike == 0 && ball == 0)
        {
            printf("아웃!\n\n");
        }
        else
        {
            printf("%d스트라이크 %d볼\n\n",
                strike, ball);
        }
    }

    return 0;
}

```
<img width="1100" height="486" alt="image" src="https://github.com/user-attachments/assets/ba740ef5-3412-4b5c-969c-5153ebd58274" />

