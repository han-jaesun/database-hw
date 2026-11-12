# Chapter 09 확장 실습 답안

> **과제:** 트랜잭션으로 데이터 정합성 지키기
> **제출 방법:** LMS에는 본인 GitHub 저장소의 `chapter09_answer.md` 파일 URL을 제출합니다.

---

## 제출 전 주의

이 파일과 캡처 화면에는 실제 비밀번호, 전체 DB 접속 URL, API Key, 개인정보를 기록하지 않습니다.

```text
GitHub 계정 또는 별칭: han-jaesun
과제 작성일: 2026-10-27
사용한 AI 도구: Claude
```

---

# 1. 시작 환경과 Chapter 07·08 기준 상태 확인

```sql
SELECT current_database();
SELECT current_user;
SELECT current_schema();
SHOW search_path;
SHOW transaction_read_only;
```

| 확인 항목 | 실제 결과 | 의미 |
| --- | --- | --- |
| `current_database()` | ai_database_book | 실습용 데이터베이스에 연결되어 있다 |
| `current_user` | hanjaesun | 스키마 생성과 데이터 변경을 할 사용자로 접속했다 |
| `current_schema()` | public | 기본 스키마는 public이지만 이번 SQL은 transaction_lab., course_project.를 명시하므로 영향이 없다 |
| `search_path` | public, "$user" | 스키마 이름 없이 쓴 테이블은 public에서 찾는다 |
| `transaction_read_only` | off | 쓰기 가능한 연결이라 COMMIT·ROLLBACK 실습이 가능하다 |

Chapter 07·08 기준값:

```text
students = 3
instructors = 2
courses = 3
enrollments = 5
전체 recorded_amount = 590000
활성 = 3 / 340000
취소 제외 = 4 / 440000
```

### 기준 상태가 다르면 Chapter 09를 계속 진행하면 안 되는 이유

```text
Chapter 09의 좌석·신청·결제 테이블은 course_project의 학생과 강의를 FK로 참조하고, 신청 금액도 강의 가격을 가져와 계산한다.
기준 상태가 다르면 실습 결과 숫자가 달라지고, 결과가 틀렸을 때 그것이 트랜잭션 문제인지 이전 Chapter 데이터 문제인지 구분할 수 없다.
실제로 Chapter 07 실습 중 기준 데이터 없이 03 파일을 실행해 변경이 일어나지 않은 경험이 있었다.
그래서 기대값을 바꾸지 않고, 기준이 다르면 원인을 찾아 Chapter 07 기준 상태를 복원한 뒤 다시 검사하고 시작한다.
```

---

# 2. `transaction_lab` 스키마 생성

실행 파일:

```text
code/chapter09/01_transaction_lab_schema.sql
```

## 2-1. 생성 전 예상

```text
생성될 스키마: transaction_lab
생성될 테이블 3개: course_inventory, enrollments, payments

course_inventory 한 행의 의미: 특정 강의 하나의 좌석 상태 (정원 capacity와 남은 좌석 remaining_seats)
enrollments 한 행의 의미: transaction_lab에서 만든 수강신청 사건 한 건 (학생, 강의, 상태, 신청 금액)
payments 한 행의 의미: 특정 lab 신청 한 건에 연결된 결제 기록 한 건 (신청당 결제는 하나, UNIQUE)
```

## 2-2. 생성 결과

```text
통과 메시지: Chapter 09 transaction lab schema validation passed
```

### Chapter 07·08의 `course_project`와 별도 `transaction_lab`을 사용하는 이유

```text
트랜잭션 실습은 좌석을 줄이고, 신청과 결제를 만들고, 일부러 ROLLBACK하는 등 데이터를 계속 바꾼다.
이 실험을 course_project에서 하면 Chapter 07·08에서 검증한 기준 데이터(5건, 590000원)가 망가질 수 있다.
별도 스키마를 쓰면 앞 장 데이터를 보호하면서 같은 학생·강의 ID를 그대로 참조할 수 있고, 실험이 꼬이면 transaction_lab만
따로 초기화할 수 있다. 삭제 범위도 transaction_lab 안으로 제한된다.
```

### 증거 화면

```text
assignments/chapter09/images/step02_schema.png
```

![transaction_lab 구조](./images/step02_schema.png)

---

# 3. 초기 좌석과 기준 데이터 입력

실행 파일:

```text
code/chapter09/02_transaction_lab_seed.sql
```

## 3-1. 실행 전 예상

```text
course 301 remaining_seats 예상: 2 (capacity 2)
course 302 remaining_seats 예상: 1 (capacity 1)
course 303 remaining_seats 예상: 1 (capacity 1)
lab enrollments 예상 행 수: 0
payments 예상 행 수: 0
```

## 3-2. 실제 결과

```text
course 301 remaining_seats: 2
course 302 remaining_seats: 1
course 303 remaining_seats: 1
lab enrollments 행 수: 0
payments 행 수: 0
통과 메시지: Chapter 09 transaction lab seed validation passed
```

### 예상과 실제가 다른 경우 원인

```text
실습 안내문에는 초기 상태가 3 / 0 / 0으로 적혀 있었지만, 실제 02 seed 파일과 그 안의 검증 조건은
301 = 2/2, 302 = 1/1, 303 = 1/1이다. 이후 03~05 실행 뒤 최종 기준(301 = 1, 302 = 0, 303 = 1)도 이 초기값에서
301 신청 1건, 302 신청 1건이 일어난 결과와 맞는다. 그래서 실행 파일의 검증 조건을 기준으로 2 / 1 / 1을 초기 상태로 판단했다.
```

---

# 4. 첫 번째 정상 COMMIT 추적

실행 파일:

```text
code/chapter09/03_commit_transaction.sql
```

## 4-1. 업무 단위 정의

```text
이 트랜잭션에서 함께 성공해야 하는 변경 1: course 301의 남은 좌석을 1 줄인다 (좌석이 있을 때만)
변경 2: 학생 101의 강의 301 신청 9001을 만든다 (금액은 강의 가격 100000)
변경 3: 신청 9001에 연결된 결제 9901을 만든다 (금액 100000)

하나라도 실패하면 전체를 취소해야 하는 이유:
좌석만 줄고 신청이 없으면 아무도 안 쓰는 좌석이 사라지고, 신청은 있는데 결제가 없으면 돈을 안 낸 수강생이 생긴다.
세 변경은 "학생 101의 수강신청"이라는 하나의 업무라서, 일부만 반영되면 좌석·신청·결제가 서로 맞지 않는 모순 상태가 된다.
```

## 4-2. 상태 변화 기록

| 시점 | course 301 남은 좌석 | lab enrollment 9001 | payment 9901 | 설명 |
| --- | ---: | --- | --- | --- |
| BEGIN 전 | 2 | 없음 | 없음 | 초기 상태 |
| 트랜잭션 내부 | 1 | 있음 (수강중, 100000) | 있음 (100000) | 현재 세션에서만 보이는 아직 확정되지 않은 변경 |
| COMMIT 후 | 1 | 있음 (수강중, 100000) | 있음 (100000) | 세 변경이 함께 확정됨 |

## 4-3. COMMIT 조건

```text
좌석 UPDATE 기대 영향 행 수: 1
실제 영향 행 수: 1
신청 생성 기대 행 수: 1
결제 생성 기대 행 수: 1
recorded_amount와 payment.amount 일치 여부: 일치 (100000 = 100000)
최종 COMMIT 판단: 좌석 1 감소, 신청 1건, 결제 1건, 금액 일치, 학생·강의 참조 유효를 모두 확인했으므로 COMMIT
통과 메시지: Chapter 09 first commit validation passed
```

### SQL 오류가 없었다는 사실만으로 COMMIT하면 안 되는 이유

```text
SQL은 문법만 맞으면 오류 없이 실행된다. 예를 들어 WHERE 조건을 잘못 써서 좌석이 2행 줄어들거나, 좌석이 0이라 UPDATE가
0행이었어도 오류는 나지 않는다. 오류가 없다는 것은 "명령이 실행됐다"는 뜻일 뿐 "업무가 올바르게 끝났다"는 뜻이 아니다.
그래서 영향 행 수가 기대값과 같은지, 여러 테이블의 최종 상태가 서로 맞는지 확인한 뒤에만 COMMIT해야 한다.
```

### 증거 화면

```text
assignments/chapter09/images/step04_commit.png
```

![첫 번째 COMMIT 결과](./images/step04_commit.png)

---

# 5. ROLLBACK으로 전체 원상복구 확인

실행 파일:

```text
code/chapter09/04_rollback_transaction.sql
```

## 5-1. ROLLBACK 전 예상

```text
트랜잭션 안에서 임시로 바뀔 값: course 302 남은 좌석 1 → 0, 학생 102의 임시 신청 9002와 결제 9902 생성
ROLLBACK 후 다시 돌아와야 할 값: course 302 남은 좌석 1, 9002·9902 없음
이미 이전 파일에서 COMMIT된 9001/9901은 유지되어야 하는가: 유지되어야 한다. ROLLBACK은 현재 트랜잭션의 변경만 취소한다.
```

## 5-2. 실제 결과

```text
ROLLBACK 후 course 301 상태: 남은 좌석 1 (03에서 COMMIT한 값 그대로)
ROLLBACK 후 course 302 상태: 남은 좌석 1 (임시 차감이 취소됨)
ROLLBACK 후 lab enrollments 행 수: 1 (9001만)
ROLLBACK 후 payments 행 수: 1 (9901만)
9001 존재 여부: 있음
9901 존재 여부: 있음
통과 메시지: Chapter 09 rollback validation passed
```

## 5-3. ROLLBACK과 IDENTITY

```text
ROLLBACK이 테이블 행 변경을 되돌리는 방식:
BEGIN 이후 아직 COMMIT하지 않은 INSERT·UPDATE·DELETE를 모두 없던 일로 만든다. 그래서 좌석 차감과 9002·9902 생성이 함께 사라진다.
이미 COMMIT된 9001·9901은 이전 트랜잭션에서 확정되었으므로 영향을 받지 않는다.

IDENTITY 자동 번호가 반드시 이전 값으로 되돌아가지는 않는 이유:
자동 번호는 여러 사용자가 동시에 써도 겹치지 않도록 한 번 나눠 준 번호를 다시 회수하지 않는 방식으로 동작한다.
트랜잭션이 취소되어도 이미 꺼내 쓴 번호는 되돌리지 않기 때문에 다음 행은 그다음 번호를 받는다.

번호가 건너뛰었다고 데이터 손상이라고 단정할 수 없는 이유:
id의 역할은 행을 겹치지 않게 구분하는 것이지 빠짐없는 순번을 보장하는 것이 아니다. Chapter 04에서도 실패한 INSERT 때문에
id가 17번을 건너뛴 적이 있었지만 데이터는 정상이었다. 손상 여부는 번호의 연속성이 아니라 행 수·금액·관계 같은 검증으로 판단한다.
(이번 04 파일은 9002처럼 id를 직접 지정해서 넣었기 때문에, 05에서 같은 9002를 다시 사용할 수 있었다.)
```

---

# 6. 두 번째 COMMIT과 좌석 부족 0행 관찰

실행 파일:

```text
code/chapter09/05_commit_and_sold_out.sql
```

## 6-1. 두 번째 정상 COMMIT

```text
생성된 enrollment id: 9002
학생 id: 103
course id: 302
recorded_amount: 120000
payment id: 9902
payment amount: 120000
```

## 6-2. 좌석 부족 시도

```text
좌석 확보 UPDATE 기대 영향 행 수: 0 (course 302는 9002 신청으로 남은 좌석이 0)
실제 영향 행 수: 0
후속 enrollment 생성 행 수: 0
후속 payment 생성 행 수: 0
통과 메시지: Chapter 09 sold-out validation passed
```

기준상 좌석 부족 시 생성되지 않아야 하는 ID:

```text
9003 → 존재하지 않음
9903 → 존재하지 않음
```

### `UPDATE 0`이 SQL 실패가 아니라 업무상 실패일 수 있는 이유

```text
"남은 좌석이 0보다 클 때만 1 줄여라"라는 조건부 UPDATE는, 좌석이 없으면 조건에 맞는 행이 없어 0행을 바꾸고 정상 종료된다.
문법도 맞고 오류 메시지도 없다. 하지만 업무적으로는 좌석을 확보하지 못한 것, 즉 신청 실패다.
그래서 오류 여부가 아니라 영향 행 수를 보고 성공·실패를 판단해야 한다.
```

### 영향 행 수가 0인데 신청과 결제를 계속 생성하면 어떤 정합성 문제가 생기나요?

```text
좌석은 확보되지 않았는데 신청 9003과 결제 9903만 생긴다. 그러면 정원 1명인 강의 302에 수강생이 2명이 되고,
남은 좌석 0과 활성 신청 수 2가 서로 맞지 않는다. 결제까지 받았는데 실제로 들을 자리가 없는 학생이 생기는 셈이다.
그래서 좌석 UPDATE가 0행이면 후속 INSERT를 하지 않고 ROLLBACK해야 한다.
```

---

# 7. 주 실습 최종 정합성 검증

실행 파일:

```text
code/chapter09/06_transaction_validation.sql
```

## 7-1. lab 최종 상태

| 항목 | 기대값 | 실제값 | 일치? |
| --- | ---: | ---: | --- |
| course_inventory 행 수 | 3 | 3 | 일치 |
| lab enrollments 행 수 | 2 | 2 | 일치 |
| payments 행 수 | 2 | 2 | 일치 |
| course 301 remaining | 1 | 1 | 일치 |
| course 302 remaining | 0 | 0 | 일치 |
| course 303 remaining | 1 | 1 | 일치 |

## 7-2. 주요 행

```text
9001 = student 101 / course 301 / amount 100000 / payment 9901
실제: student 101 / course 301 / amount 100000 / payment 9901

9002 = student 103 / course 302 / amount 120000 / payment 9902
실제: student 103 / course 302 / amount 120000 / payment 9902

9003·9903 = 존재하지 않아야 함
실제: 둘 다 존재하지 않음
```

## 7-3. 보호 대상 확인

```text
course_project.enrollments 행 수: 5
전체 recorded_amount: 590000
활성 건수/금액: 3 / 340000
취소 제외 건수/금액: 4 / 440000
최종 메시지: Chapter 09 main transaction validation passed
```

### transaction_lab 실습 후에도 course_project 기준 상태를 다시 검사하는 이유

```text
transaction_lab은 course_project의 학생과 강의를 참조하기 때문에, 실습 SQL이 실수로 course_project를 바꿨을 가능성을 배제해야 한다.
완료 기준은 "새 실습 결과가 맞다"만이 아니라 "기존 프로젝트 데이터도 그대로다"까지 포함한다.
격리해서 실험했다는 주장도 실제로 다시 검사해야 증명된다.
```

### 증거 화면

```text
assignments/chapter09/images/step07_validation.png
```

![최종 정합성 검증](./images/step07_validation.png)

---

# 8. ACID를 이번 실습으로 설명

```text
Atomicity(원자성):
좌석 1 감소, 신청 9001 생성, 결제 9901 생성은 전부 반영되거나 전부 취소된다. 04 실습에서 ROLLBACK하자 302 좌석 차감과
9002·9902 생성이 한꺼번에 사라진 것이 예다.

Consistency(일관성):
트랜잭션 전후 모두 프로젝트 규칙을 만족해야 한다. 남은 좌석은 0 이상이고 정원 이하, 신청 금액과 결제 금액은 같고,
사용된 좌석 수와 활성 신청 수가 맞아야 한다. 좌석이 0인 강의 302에 9003이 생기지 않은 것이 일관성을 지킨 결과다.

Isolation(격리성):
두 사람이 동시에 강의 303의 마지막 좌석을 신청해도 서로의 작업이 섞이지 않고 정해진 순서로 처리된다.
먼저 행을 잠근 세션이 끝날 때까지 다른 세션은 기다리므로, 한 좌석이 두 번 팔리지 않는다.

Durability(지속성):
COMMIT한 9001·9901은 그 뒤 04 실습에서 ROLLBACK을 해도, DBeaver를 다시 연결해도 그대로 남아 있었다.
COMMIT된 결과는 확정된 상태로 유지된다.
```

### Atomicity와 Consistency가 같은 뜻이 아닌 이유

```text
Atomicity는 "전부 아니면 전무"라는 묶음 방식일 뿐, 묶은 내용이 옳은지는 보장하지 않는다. 잘못된 SQL 세 개를 한 트랜잭션으로
묶어 COMMIT하면 세 개가 모두 잘못된 채로 확정된다. Consistency는 결과가 업무 규칙에 맞는지에 관한 것이라,
제약조건·조건부 UPDATE·영향 행 수 확인·검증 SELECT가 함께 있어야 지켜진다.
```

---

# 9. 선택 실습 — 두 세션 Lock 대기 관찰

실행 파일:

```text
code/chapter09/07_concurrency_two_sessions.sql
```

## 9-1. 내 환경

```text
실제 두 세션 실습 수행 / 절차 분석만 수행: 절차 분석만 수행
transaction_isolation: read committed (PostgreSQL 기본값)
lock_timeout: 5s (SET LOCAL lock_timeout = '5s'로 무한 대기 방지)
```

## 9-2. 시간 순서 기록

| 순서 | Session A | Session B | 관찰 |
| ---: | --- | --- | --- |
| 1 | BEGIN → course 303 행 `SELECT ... FOR UPDATE` | – | A가 303 좌석 행의 잠금을 획득한다 |
| 2 | (COMMIT하지 않고 대기) | BEGIN → 같은 303 행 `SELECT ... FOR UPDATE` | B는 A의 잠금이 풀릴 때까지 기다린다 |
| 3 | – | (계속 대기) | 5초가 지나면 lock_timeout 오류로 B가 실패한다 |
| 4 | COMMIT 또는 ROLLBACK | 대기 중이었다면 잠금을 얻고 최신 좌석 값을 읽는다 | A가 끝난 뒤에만 B가 진행된다 |

```text
먼저 Lock을 획득한 세션: Session A
대기한 세션: Session B
A가 COMMIT/ROLLBACK한 뒤 B에서 일어난 일: B가 잠금을 얻고 A가 확정한 최신 남은 좌석을 읽는다.
A가 마지막 좌석을 가져갔다면 B는 남은 좌석 0을 보고 신청에 실패해야 한다.
```

### Lock 대기와 Deadlock의 차이

```text
Lock 대기는 A가 가진 잠금이 풀리기를 B가 한쪽 방향으로 기다리는 것이라, A가 끝나면 자연스럽게 풀린다.
Deadlock은 A는 B의 잠금을, B는 A의 잠금을 서로 기다리는 순환이라 둘 다 영원히 진행할 수 없다.
그래서 이번 실습처럼 한쪽만 기다리는 상황을 Deadlock이라고 부르지 않는다.
```

### `SELECT ... FOR UPDATE`가 모든 UPDATE 앞에 항상 필요한 것은 아닌 이유

```text
UPDATE 자체도 바꾸는 행에 잠금을 건다. "남은 좌석이 0보다 클 때만 1 줄여라"처럼 조건부 UPDATE 한 문장으로 확인과 변경을
동시에 하고, 영향 행 수로 성공 여부를 판단할 수 있다면 미리 FOR UPDATE로 잠글 필요가 없다.
FOR UPDATE는 값을 먼저 읽고 그 값으로 여러 단계의 판단을 해야 할 때, 그사이 다른 사람이 값을 바꾸지 못하게 할 때 유용하다.
```

---

# 10. 선택 실습 — 취소와 좌석 복구

실행 파일:

```text
code/chapter09/08_cancel_and_restore.sql
```

```text
9001 취소 성공 행 수: 1 (수강중 → 취소)
course 301 좌석 변화: 1 → 2
같은 취소를 다시 시도한 행 수: 0 (이미 취소 상태라 "수강중일 때만" 조건에 맞지 않음)
두 번째 좌석 복구 행 수: 0 (취소에 성공한 행이 없으므로 복구할 강의도 없음)
payment 9901 유지 여부: 유지 (결제 기록은 삭제하지 않음)
최종 ROLLBACK 후 원상복구 여부: 복구됨 (9001 수강중, course 301 남은 좌석 1)
통과 메시지: Chapter 09 cancel rollback validation passed
```

### 같은 취소를 두 번 처리해도 좌석이 두 번 증가하지 않아야 하는 이유

```text
사용자가 취소 버튼을 두 번 누르거나 같은 요청이 두 번 들어올 수 있다. 그때마다 좌석을 1씩 늘리면 남은 좌석이 정원보다 커지고,
실제로는 없는 자리가 팔리게 된다. 그래서 "수강중인 신청을 취소로 바꾸는 데 성공한 경우에만" 그 강의의 좌석을 복구하도록,
성공한 변경의 결과만 다음 변경에 연결해야 한다. 두 번째 취소는 0행이므로 좌석 복구도 0행이 된다.
```

---

# 11. 선택 실습 — 오류 상태와 SAVEPOINT

실행 파일:

```text
code/chapter09/09_error_and_savepoint.sql
```

## 11-1. 일반 오류 후 트랜잭션 상태

```text
발생시킨 오류: 학생 101이 이미 수강중인 강의 301에 신청 9003을 한 번 더 넣어 활성 신청 중복 오류(uq_transaction_enrollments_active)를 냄
오류 이후 다음 SQL 실행 결과: current transaction is aborted, commands ignored until end of transaction block
전체 ROLLBACK이 필요한 이유: PostgreSQL은 트랜잭션 안에서 오류가 나면 그 트랜잭션을 실패 상태로 표시하고 이후 명령을 모두 무시한다.
                         ROLLBACK으로 트랜잭션을 끝내야 다시 SQL을 실행할 수 있다.
                         (Chapter 05 실습 중 DBeaver에서 실제로 이 오류를 겪고 ROLLBACK으로 해결했다.)
```

## 11-2. SAVEPOINT 사용

```text
SAVEPOINT 이름: before_duplicate_enrollment
오류 발생 위치: SAVEPOINT 이후 course 301 좌석을 임시로 1 줄이고, 중복 신청 INSERT를 실행한 시점
ROLLBACK TO SAVEPOINT 후 상태: SAVEPOINT 이후의 좌석 차감과 실패한 INSERT가 취소되고, 트랜잭션은 다시 사용 가능한 상태가 됨
이후 계속 실행할 수 있었는가: 있었다. 같은 트랜잭션 안에서 다음 SQL을 실행할 수 있었고, 마지막에 ROLLBACK으로 실습 전 상태로 되돌렸다.
```

### SAVEPOINT가 전체 ROLLBACK과 다른 점

```text
전체 ROLLBACK은 BEGIN 이후 모든 변경을 취소하고 트랜잭션을 끝낸다. ROLLBACK TO SAVEPOINT는 표시해 둔 지점 이후만 취소하고
트랜잭션은 계속 이어 간다. 다만 SAVEPOINT는 오류를 무시하는 기능이 아니라, 그 지점까지 되돌려도 업무가 여전히 의미 있는지
먼저 설계한 뒤에 써야 한다.
```

---

# 12. 개인 프로젝트 트랜잭션 시나리오 설계

학기 프로젝트를 물류대행(3PL) 창고 관리 서비스로 정하고, 그 핵심 업무인 주문 출고를 트랜잭션으로 설계했다.
(가정한 테이블: `orders`(주문), `inventories`(창고별 상품 재고), `outbound_shipments`(출고 기록), `inventory_movements`(재고 변동 이력))

## 12-1. 업무 정의

```text
시나리오 ID: P09-T01
업무 이름: 주문 출고 처리
사용자 행동: 창고 직원이 화주 쇼핑몰의 "출고대기" 주문 1건(상품 A 2개)을 포장해 출고 처리 버튼을 누른다.
왜 하나의 트랜잭션이어야 하는가:
주문 상태 변경, 재고 차감, 출고 기록, 재고 변동 이력은 "이 주문이 나갔다"는 하나의 사실이다. 재고만 줄고 출고 기록이 없으면 물건이 사라진 것처럼 보이고,
출고 기록만 있고 재고가 그대로면 실제보다 재고가 많아 다음 주문을 받을 수 없는 물건을 받게 된다. 전부 반영되거나 전부 취소되어야 한다.
```

## 12-2. 트랜잭션 설계표

| 항목 | 내 설계 |
| --- | --- |
| BEGIN 전 확인 상태 | 주문 상태가 "출고대기"인지, 해당 창고에 상품 A 재고가 주문 수량 이상인지 |
| 잠금/경쟁 가능 데이터 | 같은 상품 A의 재고 행. 두 주문이 마지막 재고를 동시에 출고하려 할 수 있음. 조건부 UPDATE로 확인과 차감을 한 번에 처리 |
| 변경 1 | 주문 상태를 "출고대기 → 출고완료"로 변경 (`WHERE id = ? AND status = '출고대기'`) |
| 기대 영향 행 수 | 1 |
| 변경 2 | 재고 차감 `quantity = quantity - 2` (`WHERE 창고·상품 일치 AND quantity >= 2`) |
| 기대 영향 행 수 | 1 |
| 추가 변경 | 출고 기록 1행 INSERT, 재고 변동 이력(-2) 1행 INSERT |
| COMMIT 전 검증 | 재고가 0 이상인지, 출고 수량 = 주문 수량인지, 주문이 출고완료이고 출고 기록이 정확히 1건인지 |
| COMMIT 조건 | 변경 1·2가 각각 정확히 1행, INSERT 2건 성공, 위 검증 모두 통과 |
| ROLLBACK 조건 | 변경 1 또는 2가 0행, INSERT 오류, 검증 중 하나라도 불일치 |

## 12-3. 실패 시나리오

```text
실패 1: 재고 부족
어느 단계에서 발생: 변경 2 (재고가 1개뿐이라 quantity >= 2 조건에 맞지 않아 UPDATE 0행)
남으면 안 되는 부분 상태: 주문은 출고완료인데 재고는 그대로이고 출고 기록이 없는 상태
ROLLBACK 후 기대 상태: 주문은 다시 출고대기, 재고 1개 그대로, 출고 기록·이력 없음

실패 2: 이미 취소된 주문
어느 단계에서 발생: 변경 1 (화주가 방금 주문을 취소해 status가 "취소"라 UPDATE 0행)
남으면 안 되는 부분 상태: 취소된 주문의 재고가 차감되거나 출고 기록이 생긴 상태
ROLLBACK 후 기대 상태: 주문은 취소 그대로, 재고·출고 기록·이력 변화 없음
```

## 12-4. SQL 초안

```sql
-- 아직 테이블 구현 전이라 의사 SQL에 가깝습니다.
BEGIN;

-- 변경 1: 주문 상태 (출고대기일 때만)
UPDATE orders
SET status = '출고완료'
WHERE id = 5001
  AND status = '출고대기';
-- 영향 행 수가 0이면 ROLLBACK (실패 2)

-- 변경 2: 재고 차감 (재고가 충분할 때만)
UPDATE inventories
SET quantity = quantity - 2
WHERE warehouse_id = 1
  AND product_id = 701
  AND quantity >= 2;
-- 영향 행 수가 0이면 ROLLBACK (실패 1)

-- 추가 변경: 출고 기록과 재고 변동 이력
INSERT INTO outbound_shipments (order_id, shipped_at)
VALUES (5001, CURRENT_TIMESTAMP);

INSERT INTO inventory_movements (warehouse_id, product_id, change_qty, reason, order_id)
VALUES (1, 701, -2, '출고', 5001);

-- COMMIT 전 검증
SELECT quantity FROM inventories WHERE warehouse_id = 1 AND product_id = 701;  -- 0 이상인지
SELECT COUNT(*) FROM outbound_shipments WHERE order_id = 5001;                  -- 정확히 1건인지

COMMIT;   -- 모든 조건이 맞을 때만, 아니면 ROLLBACK;
```

---

# 13. AI를 트랜잭션 리뷰어로 활용

## 13-1. 사용한 프롬프트

```text
다음 PostgreSQL 트랜잭션 설계를 검토해 주세요.
바로 완성 SQL부터 만들지 말고 먼저 다음 순서로 검토해 주세요.
1. 하나의 업무 단위가 어디까지인지
2. 트랜잭션 전에 확인할 상태
3. 동시 실행 시 경쟁할 수 있는 행
4. Lock 또는 조건부 UPDATE가 필요한지
5. 각 변경의 기대 영향 행 수
6. 여러 테이블 사이의 최종 정합성 검증
7. COMMIT 조건
8. ROLLBACK 조건
9. 실패가 SQL 오류가 아니라 업무상 0행일 수 있는 지점
그 다음 PostgreSQL 초안을 제안해 주세요.
[12-1 업무 설명] [가정한 테이블 4개] [12-4 트랜잭션 초안]
```

## 13-2. AI 제안 검토

| AI 제안 | 수용 / 수정 / 보류 / 거절 | 실제 또는 논리 검증 | 판단 이유 |
| --- | --- | --- | --- |
| 주문 상태 확인을 재고 차감보다 먼저 하라 | 수용 | 실패 2(취소된 주문)를 따라가 보면, 재고를 먼저 줄인 뒤 주문 확인에서 실패하면 되돌릴 일이 생긴다 | 초안을 주문 상태 변경 → 재고 차감 순서로 바꿨다. 취소된 주문이면 첫 단계에서 바로 멈춘다 |
| 각 UPDATE 뒤 영향 행 수가 1이 아니면 즉시 ROLLBACK하는 분기를 넣어라 | 수용 | 실패 1에서 UPDATE 0행인데 INSERT를 계속하면 재고는 그대로인데 출고 기록만 생긴다 | Chapter 09 05 실습의 9003·9903과 같은 문제라 그대로 반영했다 |
| 재고가 모자라면 있는 만큼만 부분 출고하라 | 거절 | 요구사항에 부분 출고 정책이 없다 | 부분 출고를 허용할지는 화주와 정해야 할 미확정 정책이다. 지금은 재고 부족이면 출고 전체를 실패시킨다 |
| 동시 출고 충돌을 막기 위해 격리 수준을 SERIALIZABLE로 올려라 | 보류 | 조건부 UPDATE `quantity >= 2` 한 문장으로 확인과 차감을 동시에 하면 마지막 재고를 두 번 출고할 수 없다 | 현재 설계로 충분하고, 격리 수준 조정은 아직 깊게 배우지 않았다. 실제 충돌 테스트 후 다시 검토한다 |

### AI SQL에서 확인한 가장 중요한 위험

```text
AI 초안 중 하나는 BEGIN과 COMMIT으로 감싸기만 하고 UPDATE의 영향 행 수를 확인하지 않았다. 그러면 재고가 부족해 UPDATE가 0행이어도
출고 기록과 이력 INSERT가 그대로 실행되고 COMMIT된다. 재고는 줄지 않았는데 출고된 것으로 기록되는 정합성 오류가 확정된다.
```

### "오류가 없으면 COMMIT"만으로 부족한 이유

```text
재고 부족, 이미 취소된 주문처럼 업무상 실패는 SQL 오류가 아니라 영향 행 수 0으로 나타난다. 오류만 기준으로 하면 이런 실패를 성공으로
착각하고 COMMIT하게 된다. 영향 행 수와 여러 테이블의 최종 상태를 확인한 뒤에만 COMMIT해야 한다.
```

---

# 14. 최종 성찰

```text
1. 트랜잭션은 여러 SQL을 단순히 묶는 것이 아니라
   하나의 업무가 성공하거나 실패하는 경계를 정하고, 영향 행 수와 여러 테이블의 최종 상태가 모두 맞을 때만 확정하는 방법 이다.

2. ROLLBACK이 필요한 대표 상황은
   좌석이 없어 좌석 UPDATE가 0행이었는데 신청과 결제가 만들어지려는 것처럼, 업무의 일부만 성공해 데이터가 서로 맞지 않게 될 때 이다.

3. 조건부 UPDATE의 영향 행 수가 중요한 이유는
   좌석·재고가 부족할 때 SQL은 오류 없이 0행으로 끝나기 때문에, 행 수를 봐야만 업무가 실패했다는 것을 알 수 있기 때문 이다.

4. 제약조건이 있어도 트랜잭션이 필요한 이유는
   제약조건은 한 행 한 행의 값이 규칙에 맞는지만 막고, "좌석 감소·신청·결제가 함께 일어났는가"처럼 여러 변경이
   함께 맞아야 하는 관계는 지켜 주지 못하기 때문 이다.

5. Lock이 필요한 이유는
   두 사람이 동시에 마지막 좌석이나 마지막 재고를 가져가려 할 때, 한 사람씩 순서대로 처리해 같은 자원이 두 번 팔리지 않게 하기 위해서 이다.

6. AI가 만든 트랜잭션 SQL을 검토할 때 가장 먼저 확인할 것은
   UPDATE가 0행일 때 후속 INSERT를 멈추고 ROLLBACK하는지, 즉 오류가 아니라 영향 행 수로 성공을 판단하는지 이다.
```

---

# 15. 제출 체크리스트

- [x] `chapter09_answer.md`를 본인 저장소에 만들었다.
- [x] Chapter 07·08 기준 상태를 확인했다.
- [x] `transaction_lab` 스키마와 초기 데이터를 만들었다.
- [x] 정상 COMMIT의 전·중·후 상태를 기록했다.
- [x] ROLLBACK 후 부분 변경이 남지 않는지 확인했다.
- [x] ROLLBACK과 IDENTITY 번호의 차이를 설명했다.
- [x] 좌석 부족 시 영향 행 수 0을 관찰했다.
- [x] 영향 행 수 0일 때 후속 행이 생성되지 않음을 확인했다.
- [x] `06_transaction_validation.sql` 최종 검증을 통과했다.
- [x] `course_project`가 변경되지 않았음을 확인했다.
- [x] ACID를 이번 실습 사례로 설명했다.
- [x] Lock 실습 또는 두 세션 절차 분석을 수행했다.
- [x] 개인 프로젝트 트랜잭션 시나리오를 작성했다.
- [x] AI 제안의 COMMIT/ROLLBACK/영향 행 수 검증을 확인했다.
- [x] 핵심 캡처는 3~4장 정도로 정리했다.
- [x] 캡처에 비밀번호·개인정보가 없다.
- [x] GitHub 웹에서 Markdown과 이미지가 정상적으로 보인다.
- [x] 최종 파일을 commit/push했다.

---

# 16. LMS 제출 URL

```text
https://github.com/han-jaesun/database-hw/blob/main/assignments/chapter09/chapter09_answer.md
```
