프롬프트 : C언어로 sqlite를 이용해서 DB를 생성하고 구현해야해. 그러나 시나리오는 도서 대여 관리로 하고, 3명 이상의 데이터를 포함하고 검색 기능도 있어야 해

```c

#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "sqlite3.h"

sqlite3* db = NULL;

/* SQL 실행 함수 */
int executeSQL(const char* sql)
{
    char* errorMessage = NULL;

    int result = sqlite3_exec(db, sql, NULL, NULL, &errorMessage);

    if (result != SQLITE_OK)
    {
        printf("SQL 오류: %s\n",
            errorMessage ? errorMessage : sqlite3_errmsg(db));

        sqlite3_free(errorMessage);
        return 0;
    }

    return 1;
}

/* 데이터베이스 및 초기 데이터 생성 */
void initDB(void)
{
    const char* schema =
        "PRAGMA foreign_keys = ON;"

        "CREATE TABLE IF NOT EXISTS MEMBER ("
        " member_id INTEGER PRIMARY KEY AUTOINCREMENT,"
        " name TEXT NOT NULL,"
        " phone TEXT NOT NULL UNIQUE"
        ");"

        "CREATE TABLE IF NOT EXISTS BOOK ("
        " book_id INTEGER PRIMARY KEY AUTOINCREMENT,"
        " title TEXT NOT NULL,"
        " author TEXT NOT NULL,"
        " is_rented INTEGER NOT NULL DEFAULT 0"
        ");"

        "CREATE TABLE IF NOT EXISTS RENTAL ("
        " rental_id INTEGER PRIMARY KEY AUTOINCREMENT,"
        " member_id INTEGER NOT NULL,"
        " book_id INTEGER NOT NULL,"
        " rental_date TEXT NOT NULL,"
        " return_date TEXT,"
        " FOREIGN KEY(member_id) REFERENCES MEMBER(member_id),"
        " FOREIGN KEY(book_id) REFERENCES BOOK(book_id)"
        ");";

    if (!executeSQL(schema))
    {
        printf("테이블 생성에 실패했습니다.\n");
        return;
    }

    /* 회원 4명 등록 */
    const char* members =
        "INSERT OR IGNORE INTO MEMBER(name, phone) VALUES"
        "('김민수', '010-1111-1111'),"
        "('이서연', '010-2222-2222'),"
        "('박지훈', '010-3333-3333'),"
        "('최유진', '010-4444-4444');";

    if (!executeSQL(members))
        return;

    /* 도서 5권 등록: 프로그램을 재실행해도 중복 등록하지 않음 */
    const char* books =
        "INSERT INTO BOOK(title, author)"
        " SELECT '어린 왕자', '생텍쥐페리'"
        " WHERE NOT EXISTS (SELECT 1 FROM BOOK"
        " WHERE title='어린 왕자' AND author='생텍쥐페리');"

        "INSERT INTO BOOK(title, author)"
        " SELECT '데미안', '헤르만 헤세'"
        " WHERE NOT EXISTS (SELECT 1 FROM BOOK"
        " WHERE title='데미안' AND author='헤르만 헤세');"

        "INSERT INTO BOOK(title, author)"
        " SELECT '해리 포터', 'J.K. 롤링'"
        " WHERE NOT EXISTS (SELECT 1 FROM BOOK"
        " WHERE title='해리 포터' AND author='J.K. 롤링');"

        "INSERT INTO BOOK(title, author)"
        " SELECT '1984', '조지 오웰'"
        " WHERE NOT EXISTS (SELECT 1 FROM BOOK"
        " WHERE title='1984' AND author='조지 오웰');"

        "INSERT INTO BOOK(title, author)"
        " SELECT '아몬드', '손원평'"
        " WHERE NOT EXISTS (SELECT 1 FROM BOOK"
        " WHERE title='아몬드' AND author='손원평');";

    if (!executeSQL(books))
        return;

    /* 최초 실행 시 샘플 대여 기록 생성 */
    const char* sampleRental =
        "INSERT INTO RENTAL(member_id, book_id, rental_date)"
        " SELECT m.member_id, b.book_id, date('now','localtime')"
        " FROM MEMBER m CROSS JOIN BOOK b"
        " WHERE m.name='김민수' AND b.title='어린 왕자'"
        " AND NOT EXISTS (SELECT 1 FROM RENTAL r"
        " WHERE r.book_id=b.book_id AND r.return_date IS NULL);"

        "UPDATE BOOK SET is_rented=1"
        " WHERE book_id IN (SELECT book_id FROM RENTAL"
        " WHERE return_date IS NULL);";

    executeSQL(sampleRental);

    printf("데이터베이스 초기화가 완료되었습니다.\n");
}

/* 도서 전체 조회 */
void listBooks(void)
{
    sqlite3_stmt* stmt = NULL;

    const char* sql =
        "SELECT book_id, title, author, is_rented "
        "FROM BOOK ORDER BY book_id;";

    if (sqlite3_prepare_v2(db, sql, -1, &stmt, NULL) != SQLITE_OK)
    {
        printf("도서 조회 오류: %s\n", sqlite3_errmsg(db));
        return;
    }

    printf("\n========== 도서 목록 ==========\n");
    printf("%-5s %-20s %-20s %-10s\n",
        "번호", "도서명", "저자", "대여 상태");

    while (sqlite3_step(stmt) == SQLITE_ROW)
    {
        printf("%-5d %-20s %-20s %-10s\n",
            sqlite3_column_int(stmt, 0),
            (const char*)sqlite3_column_text(stmt, 1),
            (const char*)sqlite3_column_text(stmt, 2),
            sqlite3_column_int(stmt, 3) ? "대여 중" : "대여 가능");
    }

    sqlite3_finalize(stmt);
}

/* 도서 검색: 도서명 또는 저자 */
void searchBooks(void)
{
    char keyword[200];
    char pattern[402];
    sqlite3_stmt* stmt = NULL;

    printf("검색할 도서명 또는 저자: ");

    if (scanf_s(" %199[^\n]", keyword,
        (unsigned)sizeof(keyword)) != 1)
    {
        printf("검색어 입력 오류입니다.\n");
        return;
    }

    snprintf(pattern, sizeof(pattern), "%%%s%%", keyword);

    const char* sql =
        "SELECT book_id, title, author, is_rented "
        "FROM BOOK WHERE title LIKE ? OR author LIKE ? "
        "ORDER BY book_id;";

    if (sqlite3_prepare_v2(db, sql, -1, &stmt, NULL) != SQLITE_OK)
    {
        printf("검색 오류: %s\n", sqlite3_errmsg(db));
        return;
    }

    sqlite3_bind_text(stmt, 1, pattern, -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 2, pattern, -1, SQLITE_TRANSIENT);

    printf("\n========== 도서 검색 결과 ==========\n");

    int found = 0;

    while (sqlite3_step(stmt) == SQLITE_ROW)
    {
        found = 1;

        printf("번호: %d | 도서명: %s | 저자: %s | 상태: %s\n",
            sqlite3_column_int(stmt, 0),
            (const char*)sqlite3_column_text(stmt, 1),
            (const char*)sqlite3_column_text(stmt, 2),
            sqlite3_column_int(stmt, 3) ? "대여 중" : "대여 가능");
    }

    if (!found)
        printf("검색 결과가 없습니다.\n");

    sqlite3_finalize(stmt);
}

/* 회원 검색 */
void searchMembers(void)
{
    char keyword[200];
    char pattern[402];
    sqlite3_stmt* stmt = NULL;

    printf("검색할 회원 이름: ");

    if (scanf_s(" %199[^\n]", keyword,
        (unsigned)sizeof(keyword)) != 1)
    {
        printf("검색어 입력 오류입니다.\n");
        return;
    }

    snprintf(pattern, sizeof(pattern), "%%%s%%", keyword);

    const char* sql =
        "SELECT member_id, name, phone FROM MEMBER "
        "WHERE name LIKE ? ORDER BY member_id;";

    if (sqlite3_prepare_v2(db, sql, -1, &stmt, NULL) != SQLITE_OK)
    {
        printf("회원 검색 오류: %s\n", sqlite3_errmsg(db));
        return;
    }

    sqlite3_bind_text(stmt, 1, pattern, -1, SQLITE_TRANSIENT);

    printf("\n========== 회원 검색 결과 ==========\n");

    int found = 0;

    while (sqlite3_step(stmt) == SQLITE_ROW)
    {
        found = 1;

        printf("회원 번호: %d | 이름: %s | 연락처: %s\n",
            sqlite3_column_int(stmt, 0),
            (const char*)sqlite3_column_text(stmt, 1),
            (const char*)sqlite3_column_text(stmt, 2));
    }

    if (!found)
        printf("검색 결과가 없습니다.\n");

    sqlite3_finalize(stmt);
}

/* 도서 대여 */
void rentBook(void)
{
    int memberId, bookId;
    sqlite3_stmt* stmt = NULL;

    printf("회원 번호: ");

    if (scanf_s("%d", &memberId) != 1)
    {
        printf("잘못된 입력입니다.\n");
        return;
    }

    printf("도서 번호: ");

    if (scanf_s("%d", &bookId) != 1)
    {
        printf("잘못된 입력입니다.\n");
        return;
    }

    if (!executeSQL("BEGIN IMMEDIATE;"))
        return;

    /* 회원 존재 여부 확인 */
    const char* memberSQL =
        "SELECT 1 FROM MEMBER WHERE member_id=?;";

    if (sqlite3_prepare_v2(db, memberSQL, -1, &stmt, NULL) != SQLITE_OK)
    {
        executeSQL("ROLLBACK;");
        return;
    }

    sqlite3_bind_int(stmt, 1, memberId);
    int memberExists = sqlite3_step(stmt) == SQLITE_ROW;
    sqlite3_finalize(stmt);

    if (!memberExists)
    {
        printf("존재하지 않는 회원입니다.\n");
        executeSQL("ROLLBACK;");
        return;
    }

    /* 대여 가능한 도서만 상태 변경 */
    const char* bookSQL =
        "UPDATE BOOK SET is_rented=1 "
        "WHERE book_id=? AND is_rented=0;";

    if (sqlite3_prepare_v2(db, bookSQL, -1, &stmt, NULL) != SQLITE_OK)
    {
        executeSQL("ROLLBACK;");
        return;
    }

    sqlite3_bind_int(stmt, 1, bookId);

    int updateOK = sqlite3_step(stmt) == SQLITE_DONE;
    int changed = sqlite3_changes(db);

    sqlite3_finalize(stmt);

    if (!updateOK || changed != 1)
    {
        printf("도서가 없거나 이미 대여 중입니다.\n");
        executeSQL("ROLLBACK;");
        return;
    }

    /* 대여 기록 추가 */
    const char* rentalSQL =
        "INSERT INTO RENTAL(member_id, book_id, rental_date) "
        "VALUES(?, ?, date('now','localtime'));";

    if (sqlite3_prepare_v2(db, rentalSQL, -1, &stmt, NULL) != SQLITE_OK)
    {
        executeSQL("ROLLBACK;");
        printf("대여 기록 생성에 실패했습니다.\n");
        return;
    }

    sqlite3_bind_int(stmt, 1, memberId);
    sqlite3_bind_int(stmt, 2, bookId);

    int insertOK = sqlite3_step(stmt) == SQLITE_DONE;
    sqlite3_finalize(stmt);

    if (!insertOK)
    {
        executeSQL("ROLLBACK;");
        printf("대여 처리에 실패했습니다.\n");
        return;
    }

    if (executeSQL("COMMIT;"))
        printf("도서 대여가 완료되었습니다.\n");
    else
        executeSQL("ROLLBACK;");
}

/* 도서 반납 */
void returnBook(void)
{
    int bookId;
    sqlite3_stmt* stmt = NULL;

    printf("반납할 도서 번호: ");

    if (scanf_s("%d", &bookId) != 1)
    {
        printf("잘못된 입력입니다.\n");
        return;
    }

    if (!executeSQL("BEGIN IMMEDIATE;"))
        return;

    const char* rentalSQL =
        "UPDATE RENTAL SET return_date=date('now','localtime') "
        "WHERE book_id=? AND return_date IS NULL;";

    if (sqlite3_prepare_v2(db, rentalSQL, -1, &stmt, NULL) != SQLITE_OK)
    {
        executeSQL("ROLLBACK;");
        return;
    }

    sqlite3_bind_int(stmt, 1, bookId);

    int updateOK = sqlite3_step(stmt) == SQLITE_DONE;
    int changed = sqlite3_changes(db);

    sqlite3_finalize(stmt);

    if (!updateOK || changed != 1)
    {
        printf("대여 중인 도서가 없습니다.\n");
        executeSQL("ROLLBACK;");
        return;
    }

    const char* bookSQL =
        "UPDATE BOOK SET is_rented=0 WHERE book_id=?;";

    if (sqlite3_prepare_v2(db, bookSQL, -1, &stmt, NULL) != SQLITE_OK)
    {
        executeSQL("ROLLBACK;");
        return;
    }

    sqlite3_bind_int(stmt, 1, bookId);

    int bookUpdated = sqlite3_step(stmt) == SQLITE_DONE;
    sqlite3_finalize(stmt);

    if (!bookUpdated)
    {
        executeSQL("ROLLBACK;");
        printf("반납 처리에 실패했습니다.\n");
        return;
    }

    if (executeSQL("COMMIT;"))
        printf("도서 반납이 완료되었습니다.\n");
    else
        executeSQL("ROLLBACK;");
}

/* 대여 기록 조회 */
void listRentals(void)
{
    sqlite3_stmt* stmt = NULL;

    const char* sql =
        "SELECT r.rental_id, m.name, b.title, "
        "r.rental_date, r.return_date "
        "FROM RENTAL r "
        "JOIN MEMBER m ON r.member_id=m.member_id "
        "JOIN BOOK b ON r.book_id=b.book_id "
        "ORDER BY r.rental_id DESC;";

    if (sqlite3_prepare_v2(db, sql, -1, &stmt, NULL) != SQLITE_OK)
    {
        printf("대여 기록 조회 오류: %s\n", sqlite3_errmsg(db));
        return;
    }

    printf("\n========== 대여 기록 ==========\n");

    int found = 0;

    while (sqlite3_step(stmt) == SQLITE_ROW)
    {
        found = 1;

        const unsigned char* returnDate =
            sqlite3_column_text(stmt, 4);

        printf("대여 번호: %d | 회원: %s | 도서: %s "
            "| 대여일: %s | 반납일: %s\n",
            sqlite3_column_int(stmt, 0),
            (const char*)sqlite3_column_text(stmt, 1),
            (const char*)sqlite3_column_text(stmt, 2),
            (const char*)sqlite3_column_text(stmt, 3),
            returnDate ? (const char*)returnDate : "미반납");
    }

    if (!found)
        printf("대여 기록이 없습니다.\n");

    sqlite3_finalize(stmt);
}

/* 메인 함수 */
int main(void)
{
    int menu;

    if (sqlite3_open("library.db", &db) != SQLITE_OK)
    {
        printf("데이터베이스 열기 실패: %s\n",
            db ? sqlite3_errmsg(db) : "메모리 할당 실패");

        if (db != NULL)
            sqlite3_close(db);

        return 1;
    }

    printf("SQLite 데이터베이스 연결 성공!\n");

    initDB();

    do
    {
        printf("\n================================\n");
        printf("      도서 대여 관리 시스템\n");
        printf("================================\n");
        printf("1. 도서 전체 조회\n");
        printf("2. 도서 검색\n");
        printf("3. 회원 검색\n");
        printf("4. 도서 대여\n");
        printf("5. 도서 반납\n");
        printf("6. 대여 기록 조회\n");
        printf("0. 종료\n");
        printf("--------------------------------\n");
        printf("메뉴 선택: ");

        if (scanf_s("%d", &menu) != 1)
        {
            int ch;

            while ((ch = getchar()) != '\n' && ch != EOF)
            {
            }

            printf("숫자를 입력해 주세요.\n");
            menu = -1;
            continue;
        }

        switch (menu)
        {
        case 1:
            listBooks();
            break;

        case 2:
            searchBooks();
            break;

        case 3:
            searchMembers();
            break;

        case 4:
            rentBook();
            break;

        case 5:
            returnBook();
            break;

        case 6:
            listRentals();
            break;

        case 0:
            printf("프로그램을 종료합니다.\n");
            break;

        default:
            printf("잘못된 메뉴입니다. 다시 선택해 주세요.\n");
            break;
        }

    } while (menu != 0);

    sqlite3_close(db);
    db = NULL;

    return 0;
}

```
<img width="1147" height="764" alt="image" src="https://github.com/user-attachments/assets/c474a86a-55f3-4cb0-bf93-0d9a51646c57" />
