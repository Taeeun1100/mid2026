```c

#include "raylib.h"
#include <math.h>

#define SCREEN_WIDTH 1000
#define SCREEN_HEIGHT 700

#define ROWS 6
#define COLS 10
#define BLOCK_COUNT (ROWS * COLS)

#define BALL_RADIUS 9.0f

typedef enum
{
    TITLE,
    PLAYING,
    GAME_OVER,
    WIN
} GameState;

typedef struct
{
    Rectangle rect;
    Color color;
    int active;
} Block;

static Color rainbowColors[ROWS] =
{
    {255, 65, 85, 255},
    {255, 145, 45, 255},
    {255, 220, 55, 255},
    {65, 220, 115, 255},
    {65, 150, 255, 255},
    {175, 90, 255, 255}
};

static Block blocks[BLOCK_COUNT];

static Vector2 ball;
static Vector2 ballVelocity;
static Rectangle paddle;

static GameState state = TITLE;

static int score = 0;
static int lives = 3;

void ResetBall(void)
{
    ball.x = SCREEN_WIDTH / 2.0f;
    ball.y = SCREEN_HEIGHT - 100.0f;

    ballVelocity.x = 250.0f;
    ballVelocity.y = -300.0f;
}

void CreateBlocks(void)
{
    const float blockWidth = 84.0f;
    const float blockHeight = 25.0f;
    const float gap = 7.0f;

    float totalWidth =
        COLS * blockWidth + (COLS - 1) * gap;

    float startX = (SCREEN_WIDTH - totalWidth) / 2.0f;

    int index = 0;

    for (int row = 0; row < ROWS; row++)
    {
        for (int col = 0; col < COLS; col++)
        {
            blocks[index].rect = (Rectangle)
            {
                startX + col * (blockWidth + gap),
                100.0f + row * (blockHeight + gap),
                blockWidth,
                blockHeight
            };

            blocks[index].color = rainbowColors[row];
            blocks[index].active = 1;

            index++;
        }
    }
}

void ResetGame(void)
{
    score = 0;
    lives = 3;

    paddle = (Rectangle)
    {
        SCREEN_WIDTH / 2.0f - 65.0f,
        SCREEN_HEIGHT - 45.0f,
        130.0f,
        15.0f
    };

    ResetBall();
    CreateBlocks();

    state = TITLE;
}

void UpdatePaddle(void)
{
    float dt = GetFrameTime();
    float speed = 520.0f;

    if (IsKeyDown(KEY_LEFT))
        paddle.x -= speed * dt;

    if (IsKeyDown(KEY_RIGHT))
        paddle.x += speed * dt;

    if (IsMouseButtonDown(MOUSE_BUTTON_LEFT))
        paddle.x = GetMouseX() - paddle.width / 2.0f;

    if (paddle.x < 0)
        paddle.x = 0;

    if (paddle.x + paddle.width > SCREEN_WIDTH)
        paddle.x = SCREEN_WIDTH - paddle.width;
}

void UpdateBall(void)
{
    float dt = GetFrameTime();

    Vector2 previousBall = ball;

    ball.x += ballVelocity.x * dt;
    ball.y += ballVelocity.y * dt;

    // 좌우 벽 충돌
    if (ball.x - BALL_RADIUS <= 0)
    {
        ball.x = BALL_RADIUS;
        ballVelocity.x = fabsf(ballVelocity.x);
    }

    if (ball.x + BALL_RADIUS >= SCREEN_WIDTH)
    {
        ball.x = SCREEN_WIDTH - BALL_RADIUS;
        ballVelocity.x = -fabsf(ballVelocity.x);
    }

    // 천장 충돌
    if (ball.y - BALL_RADIUS <= 0)
    {
        ball.y = BALL_RADIUS;
        ballVelocity.y = fabsf(ballVelocity.y);
    }

    // 패들 충돌
    if (ballVelocity.y > 0 &&
        CheckCollisionCircleRec(ball, BALL_RADIUS, paddle))
    {
        float center = paddle.x + paddle.width / 2.0f;

        float hit =
            (ball.x - center) / (paddle.width / 2.0f);

        if (hit < -1.0f) hit = -1.0f;
        if (hit > 1.0f) hit = 1.0f;

        ballVelocity.x = hit * 350.0f;

        ballVelocity.y = -sqrtf(
            400.0f * 400.0f -
            ballVelocity.x * ballVelocity.x
        );

        ball.y = paddle.y - BALL_RADIUS;
    }

    // 블록 충돌
    for (int i = 0; i < BLOCK_COUNT; i++)
    {
        if (!blocks[i].active)
            continue;

        if (CheckCollisionCircleRec(
            ball, BALL_RADIUS, blocks[i].rect))
        {
            blocks[i].active = 0;
            score += 10;

            Rectangle r = blocks[i].rect;

            if (previousBall.y + BALL_RADIUS <= r.y ||
                previousBall.y - BALL_RADIUS >=
                    r.y + r.height)
            {
                ballVelocity.y *= -1.0f;
            }
            else
            {
                ballVelocity.x *= -1.0f;
            }

            break;
        }
    }

    // 공을 놓쳤을 때
    if (ball.y - BALL_RADIUS > SCREEN_HEIGHT)
    {
        lives--;

        if (lives <= 0)
        {
            state = GAME_OVER;
        }
        else
        {
            ResetBall();
        }
    }

    // 남은 블록 확인
    int remaining = 0;

    for (int i = 0; i < BLOCK_COUNT; i++)
    {
        if (blocks[i].active)
        {
            remaining++;
        }
    }

    if (remaining == 0)
        state = WIN;
}

void DrawBackground(void)
{
    ClearBackground((Color){12, 15, 35, 255});

    for (int i = 0; i < 20; i++)
    {
        int x = (i * 137 + 43) % SCREEN_WIDTH;
        int y = (i * 83 + 25) % SCREEN_HEIGHT;

        DrawCircle(x, y, 2, Fade(SKYBLUE, 0.35f));
    }
}

void DrawCenteredText(
    const char *text, int y, int fontSize, Color color)
{
    int width = MeasureText(text, fontSize);

    DrawText(
        text,
        (SCREEN_WIDTH - width) / 2,
        y,
        fontSize,
        color
    );
}

void DrawGame(void)
{
    // 블록
    for (int i = 0; i < BLOCK_COUNT; i++)
    {
        if (!blocks[i].active)
            continue;

        DrawRectangleRounded(
            blocks[i].rect, 0.22f, 6, blocks[i].color);

        Rectangle shine =
        {
            blocks[i].rect.x + 3,
            blocks[i].rect.y + 3,
            blocks[i].rect.width - 6,
            5
        };

        DrawRectangleRounded(
            shine, 0.5f, 4, Fade(WHITE, 0.45f));
    }

    // 패들
    DrawRectangleRounded(
        paddle, 0.7f, 8, (Color){100, 220, 255, 255});

    DrawRectangleRounded(
        (Rectangle)
        {
            paddle.x + 8,
            paddle.y + 3,
            paddle.width - 16,
            4
        },
        0.5f, 4, WHITE);

    // 공
    DrawCircleV(ball, BALL_RADIUS + 4, Fade(SKYBLUE, 0.2f));
    DrawCircleV(ball, BALL_RADIUS, RAYWHITE);

    // 점수와 목숨
    DrawText(TextFormat("SCORE: %04i", score),
             25, 25, 25, RAYWHITE);

    DrawText(TextFormat("LIVES: %i", lives),
             SCREEN_WIDTH - 160, 25, 25, PINK);

    DrawLine(20, 65, SCREEN_WIDTH - 20, 65,
             Fade(WHITE, 0.2f));
}

int main(void)
{
    InitWindow(
        SCREEN_WIDTH,
        SCREEN_HEIGHT,
        "Rainbow Breaker - C / raylib"
    );

    SetTargetFPS(60);

    ResetGame();

    while (!WindowShouldClose())
    {
        if (state == TITLE)
        {
            UpdatePaddle();

            if (IsKeyPressed(KEY_SPACE))
                state = PLAYING;
        }
        else if (state == PLAYING)
        {
            UpdatePaddle();
            UpdateBall();
        }
        else
        {
            if (IsKeyPressed(KEY_SPACE))
                ResetGame();
        }

        BeginDrawing();

        DrawBackground();

        if (state == TITLE)
        {
            DrawGame();

            DrawRectangle(
                0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
                Fade(BLACK, 0.45f));

            DrawCenteredText(
                "RAINBOW BREAKER",
                190, 52, (Color){255, 120, 220, 255});

            DrawCenteredText(
                "BREAK ALL THE RAINBOW BLOCKS!",
                280, 21, RAYWHITE);

            DrawCenteredText(
                "Press SPACE to Start",
                365, 27, YELLOW);

            DrawCenteredText(
                "Move: LEFT / RIGHT or Mouse",
                420, 19, LIGHTGRAY);
        }
        else if (state == PLAYING)
        {
            DrawGame();
        }
        else if (state == GAME_OVER)
        {
            DrawGame();

            DrawRectangle(
                0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
                Fade(BLACK, 0.72f));

            DrawCenteredText(
                "GAME OVER", 235, 55, RED);

            DrawCenteredText(
                TextFormat("FINAL SCORE: %i", score),
                325, 30, RAYWHITE);

            DrawCenteredText(
                "Press SPACE to Restart",
                400, 24, YELLOW);
        }
        else if (state == WIN)
        {
            DrawGame();

            DrawRectangle(
                0, 0, SCREEN_WIDTH, SCREEN_HEIGHT,
                Fade(BLACK, 0.60f));

            DrawCenteredText(
                "RAINBOW MASTER!",
                230, 52, (Color){255, 220, 80, 255});

            DrawCenteredText(
                TextFormat("FINAL SCORE: %i", score),
                320, 30, RAYWHITE);

            DrawCenteredText(
                "All Blocks Destroyed!",
                370, 24, SKYBLUE);

            DrawCenteredText(
                "Press SPACE to Play Again",
                425, 22, YELLOW);
        }

        EndDrawing();
    }

    CloseWindow();

    return 0;
}

```
