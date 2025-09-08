# Linux 환경에서 MySQL 설치 및 자동 백업

## 1️⃣ MySQL 설치 (Docker 금지)

```bash
# 패키지 업데이트
sudo apt update && sudo apt upgrade -y

# MySQL 서버 설치
sudo apt install mysql-server -y

# 서비스 상태 확인
sudo systemctl status mysql

# 실행 중이 아니라면 자동 시작 등록 + 즉시 실행
sudo systemctl enable --now mysql

```

- **서비스 등록 확인**

```bash
systemctl list-unit-files | grep mysql

```

---

## 2️⃣ EMP & DEPT 테이블 생성 및 데이터 저장

```sql
-- 기존 DB 제거 후 새로 생성
DROP DATABASE IF EXISTS sqld;
CREATE DATABASE sqld DEFAULT CHARACTER SET UTF8;
USE sqld;

-- 부서 테이블 생성
CREATE TABLE DEPT (
  DEPTNO DECIMAL(10),
  DNAME  VARCHAR(14),
  LOC    VARCHAR(13)
);

-- 부서 데이터 삽입
INSERT INTO DEPT VALUES (10, 'ACCOUNTING', 'NEW YORK');
INSERT INTO DEPT VALUES (20, 'RESEARCH',   'DALLAS');
INSERT INTO DEPT VALUES (30, 'SALES',      'CHICAGO');
INSERT INTO DEPT VALUES (40, 'OPERATIONS', 'BOSTON');

-- 사원 테이블 생성
CREATE TABLE EMP (
  EMPNO     DECIMAL(4) NOT NULL,
  ENAME     VARCHAR(10),
  JOB       VARCHAR(9),
  MGR       DECIMAL(4),
  HIREDATE  DATE,
  SAL       DECIMAL(7,2),
  COMM      DECIMAL(7,2),
  DEPTNO    DECIMAL(2)
);

-- 사원 데이터 삽입
INSERT INTO EMP VALUES (7839,'KING','PRESIDENT',NULL,STR_TO_DATE('1981-11-17','%Y-%m-%d'),5000,NULL,10);
INSERT INTO EMP VALUES (7698,'BLAKE','MANAGER',7839,STR_TO_DATE('1981-05-01','%Y-%m-%d'),2850,NULL,30);
INSERT INTO EMP VALUES (7782,'CLARK','MANAGER',7839,STR_TO_DATE('1981-05-09','%Y-%m-%d'),2450,NULL,10);
INSERT INTO EMP VALUES (7566,'JONES','MANAGER',7839,STR_TO_DATE('1981-04-01','%Y-%m-%d'),2975,NULL,20);
INSERT INTO EMP VALUES (7654,'MARTIN','SALESMAN',7698,STR_TO_DATE('1981-09-10','%Y-%m-%d'),1250,1400,30);
INSERT INTO EMP VALUES (7499,'ALLEN','SALESMAN',7698,STR_TO_DATE('1981-02-11','%Y-%m-%d'),1600,300,30);
INSERT INTO EMP VALUES (7844,'TURNER','SALESMAN',7698,STR_TO_DATE('1981-08-21','%Y-%m-%d'),1500,0,30);
INSERT INTO EMP VALUES (7900,'JAMES','CLERK',7698,STR_TO_DATE('1981-12-11','%Y-%m-%d'),950,NULL,30);
INSERT INTO EMP VALUES (7521,'WARD','SALESMAN',7698,STR_TO_DATE('1981-02-23','%Y-%m-%d'),1250,500,30);
INSERT INTO EMP VALUES (7902,'FORD','ANALYST',7566,STR_TO_DATE('1981-12-11','%Y-%m-%d'),3000,NULL,20);
INSERT INTO EMP VALUES (7369,'SMITH','CLERK',7902,STR_TO_DATE('1980-12-09','%Y-%m-%d'),800,NULL,20);
INSERT INTO EMP VALUES (7788,'SCOTT','ANALYST',7566,STR_TO_DATE('1982-12-22','%Y-%m-%d'),3000,NULL,20);
INSERT INTO EMP VALUES (7876,'ADAMS','CLERK',7788,STR_TO_DATE('1983-01-15','%Y-%m-%d'),1100,NULL,20);
INSERT INTO EMP VALUES (7934,'MILLER','CLERK',7782,STR_TO_DATE('1982-01-11','%Y-%m-%d'),1300,NULL,10);

COMMIT;

-- 확인
SELECT * FROM DEPT;
SELECT * FROM EMP;

```

---

## 3️⃣ 자동화 (mysqldump + tar + cron)

### (1) mysqldump 수동 실행

```bash
# company → sqld로 맞추어 실행
mysqldump -u root -pYOURPASSWORD sqld > /root/db_$(date +%Y%m%d_%H%M%S).sql

```

---

### (2) 백업 스크립트 작성

📄 `/root/script/mysql-dump.sh`

```bash
#!/bin/bash

# DB 접속 계정 정보
DB_USER="root"
DB_PASS="ubuntu"
DB_NAME="sqld"

# 백업 디렉토리
SQL_DUMP_DIR="/root/script"
mkdir -p $SQL_DUMP_DIR

# 날짜 형식: YYYYMMDD_HHMMSS
DATE=$(date +%Y%m%d_%H%M%S)
SQL_FILE="/root/db.sql"
TAR_FILE="$SQL_DUMP_DIR/MYSQL_$DATE.tar.gz"

# 1. MySQL 덤프 생성
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME > $SQL_FILE

# 2. tar 압축
tar -czf "$TAR_FILE" $SQL_FILE

# 3. 원본 SQL 삭제 (필요시)
rm -f $SQL_FILE

# 4. 로그 출력
echo "backup_success: $TAR_FILE"

```

권한 부여:

```bash
chmod +x /root/script/mysql-dump.sh

```

---

### (3) cron 등록

```bash
crontab -e

```

- 매시 1분에 실행되도록 설정:

```
1 * * * * /root/script/mysql-dump.sh >> /root/script/backup.log 2>&1

```

📌 `>> /root/script/backup.log 2>&1`

- **표준 출력**(`stdout`)과 **표준 에러**(`stderr`)를 로그 파일에 저장
- 실패/성공 여부 확인 가능
