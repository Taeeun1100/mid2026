```c

#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define DIGITS 3

// 컴퓨터가 중복 없는 정답 생성
void makeAnswer(int answer[])
{
    int used[10] = { 0 };
    int i;
    int num;

    for (i = 0; i < DIGITS; i++)
    {
        do
        {
            num = rand() % 10;
        } while (used[num] == 1 ||
                 (i == 0 && num == 0));

        answer[i] = num;
        used[num] = 1;
    }
}

// 사용자 입력 검사
int isValid(int guess[])
{
    int used[10] = { 0 };
    int i;

    if (guess[0] == 0)
        return 0;

    for (i = 0; i < DIGITS; i++)
    {
        if (guess[i] < 0 || guess[i] > 9)
            return 0;

        if (used[guess[i]] == 1)
            return 0;

        used[guess[i]] = 1;
    }

    return 1;
}

// 스트라이크와 볼 판정
void judge(int answer[], int guess[],
           int* strike, int* ball)
{
    int i, j;

    *strike = 0;
    *ball = 0;

    for (i = 0; i < DIGITS; i++)
    {
        for (j = 0; j < DIGITS; j++)
        {
            if (answer[i] == guess[j])
            {
                if (i == j)
                    (*strike)++;
                else
                    (*ball)++;
            }
        }
    }
}

int main(void)
{
    int answer[DIGITS];
    int guess[DIGITS];

    int strike, ball;
    int attempts = 0;
    int input;
    int i;

    srand((unsigned int)time(NULL));

    makeAnswer(answer);

    printf("================================\n");
    printf("       숫자야구 게임\n");
    printf("================================\n");
    printf("서로 다른 숫자 3개를 맞혀보세요!\n");
    printf("첫 번째 숫자는 0이 될 수 없습니다.\n");
    printf("예: 123\n");
    printf("--------------------------------\n");

    while (1)
    {
        printf("\n숫자 입력: ");

        // 숫자 3자리를 정수로 입력
        if (scanf("%d", &input) != 1)
        {
            printf("잘못된 입력입니다. 숫자를 입력하세요.\n");

            while ((input = getchar()) != '\n' &&
                   input != EOF)
            {
            }

            continue;
        }

        // 세 자리 숫자인지 검사
        if (input < 100 || input > 999)
        {
            printf("세 자리 숫자를 입력하세요.\n");
            continue;
        }

        guess[0] = input / 100;
        guess[1] = (input / 10) % 10;
        guess[2] = input % 10;

        // 중복 숫자 검사
        if (!isValid(guess))
        {
            printf("중복되지 않는 숫자 3개를 입력하세요.\n");
            continue;
        }

        attempts++;

        judge(answer, guess, &strike, &ball);

        printf("결과: %d 스트라이크, %d 볼\n",
               strike, ball);

        if (strike == 3)
        {
            printf("\n정답입니다!\n");
            printf("시도 횟수: %d회\n", attempts);
            break;
        }

        if (strike == 0 && ball == 0)
        {
            printf("아웃!\n");
        }
    }

    printf("정답: %d%d%d\n",
           answer[0], answer[1], answer[2]);

    printf("게임을 종료합니다.\n");

    return 0;
}

```
