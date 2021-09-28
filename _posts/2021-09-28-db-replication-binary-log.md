---
layout: post
title: DB 리플리케이션 적용시 Binary 로그 에러 해결방법
date: 2021-09-28 13:32:20 +0300
description: DB replication에서 Master db에 쿼리 오류가 났을 떄 replicas에서 해당 로그를 건너뛰지 못하는 오류를 해결하는 방법입니다.
img: database.png
tags: [백엔드, Database]
---


## INTRO

- 현재 진행중인 [프로젝트](https://github.com/woowacourse-teams/2021-pick-git)에서 DB Replication을 적용했었다. 
  - [Replication 알아보기](https://yjksw.github.io/db-replication/
- DB replication 적용 이후 Master DB를 업그레이드 해야하는 상황에서 replicas와의 연동에 문제가 생긴적이 있었다. 이때 Master와 replicas 간의 데이터 연동 방법을 이해하고 해결한 (매우 간단한) 방법을 기록한다. 

## Master DB와 replicas 동기화 

- Master DB에 데이터를 쓰기 위해서는 replicas에서 master db 의 데이터와 연결되어 있어야 한다. 그러기 위해서 replication을 설정할 때 `show master status` 라는 명령어를 통해서 나온 `File`값과 `Position` 값을 replica db 설정시 적용해 주었다. 

```mysql
MariaDB [pickgit]> show master status;
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000008 | 68143505 |              |                  |
+--------------------+----------+--------------+------------------+
1 row in set (0.000 sec)
```

- 여기서 File은 master db의 binary 로그 파일이고 Position 값은 해당 파일의 현재 위치이다. 
- 위 log 파일에는 어떤 내용이 담겨 있을까?
  > The MariaDB binary log is a series of files that contain events. An event is a description of a modification to the contents of our database. <br>출처: Big Data and Business Intelligence
- 로그 파일에는 데이터베이스에서 일어난 `event`들에 대해서 적혀 있는데, `event`라고 하는 것은 데이터베이스의 컨텐츠에 대해 일어난 변경사항을 말한다. 

```mysql
MariaDB [pickgit]> show master status;
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000008 | 68143656 |              |                  |
+--------------------+----------+--------------+------------------+
1 row in set (0.000 sec)
```

- 실제로 테이블을 추가하는 쿼리를 날린 후 다시 확인해보니 Position값이 증가한 것을 확인할 수 있다. 
- Replicas 설정시 위 값을 지정한다는 것은 replicas에 데이터를 업데이트하는 file과 해당 file에서의 위치를 지정하는 것이다. 
- 번외로 만일 binary loggin이 비활성되어 있는 상태에서 master 데이터베이스가 실행중이었다면 `show master status;` 명령어에 나오는 값이 비어있을 것이다. 그 경우 replicas에 master의 로그파일과 position을 지정할 때 빈 스트링 ('')과 4를 지정하면 된다. 


## 문제상황 및 해결 
- 본래 사용한 MariaDB 버전은 10.1이었다. 하지만 Flyway를 적용한 이후 MariaDB를 10.4로 업그레이드 하지 않으면 적용할 수 없다는 오류가 생겼다. MariaDB 버전을 업그레이드 할 수 있는 방법을 찾아보았지만 현재 사용 중인 DB 데이터를 백업하고 삭제 후 10.4 버전을 새로 설치하여 데이터를 복원하라는 내용밖에 나오지 않았다. 
- 현재 Master 1개 slave 2개를 사용중이었기 때문에 DB 3개를 모두 삭제하고 재설치하는 것은 지나치게 많은 작업이라고 생각했다. (replication 설정, 유저 생성 및 권한 부여 등등 자잘한 설정이 많음) 따라서 Flyway가 직접 적용되는 Master DB만 수정하고 Slave DB는 기존의 것을 유지하기로 했다. 
- Master DB를 새로 구성하는 와중에 다음과 같은 문제 상황이 발생했다. 
  - 문제 상황
    - Master DB의 설정을 마치고 Slave에 Master를 지정하여 연결을 완료함 
    - Master DB의 `replication` 유저에게 외부에서 쓰기 권한을 부여하지 않은 것을 깨달음 
    - Master DB의 `replication` 유저에게 권한을 부여함
    - Slave DB에 Master DB의 데이터가 반영이 되지 않음  

- Master에서 연결된 slave hosts를 확인해 보면 잘 연결되어 있는 것을 확인할 수 있다. 
  
  ```mysql
    MariaDB [pickgit]> show slave hosts;
  +-----------+------+------+-----------+
  | Server_id | Host | Port | Master_id |
  +-----------+------+------+-----------+
  |         3 |      | 9000 |         1 |
  |         2 |      | 9000 |         1 |
  +-----------+------+------+-----------+
  2 rows in set (0.000 sec)
  ```

- Slave 의 상태를 확인해보면 `Slave_IO_State: Waiting for master to sent event` 라고 나와있는 것을 확인할 수 있다. 

<p align="center"><img width="90%" src="https://user-images.githubusercontent.com/63405904/134805936-9f3469e3-aa3e-496f-91b3-ae5b7f6e881d.png"></p>

- 위 상태의 더 아래에 `Last_Error` 와 `Last_SQL_Error`를 확인해보면 특정 쿼리에 에러가 발생했다는 로그가 출력되어 있다. 즉, `replication` 이라는 유저가 Master에서는 잘 적용이 되었지만 Slave DB에는 존재하지 않기 때문에 에러가 발생한 것이다. 해당 로그 이후에 추가 및 변경된 데이터에 대해서는 slave db에 더 이상 반영이 되지 않았다. 

- 위 문제를 해결하기 위해서는 Slave DB가 Master의 로그 파일을 읽는 Position을 위 쿼리가 실행된 이후로 옮겨서 해당 쿼리를 건너뛰어야 한다. 따라서 `show master status`를 다시 실행하여 나온 최신 position을 slave DB 설정에 넣어주어 문제를 해결했다. 여기서 주의할 점은 만일 이전에 변경된 데이터가 있다면 해당 변경 로그도 모두 건너뛰게 되니 다시 적용해주어야 한다. 

**[참고자료]**
- [https://dev.mysql.com/doc/refman/8.0/en/replication-setup-replicas.html#replication-howto-newservers](https://dev.mysql.com/doc/refman/8.0/en/replication-setup-replicas.html#replication-howto-newservers)
- [https://dev.mysql.com/doc/refman/8.0/en/replication-howto-masterstatus.html](https://dev.mysql.com/doc/refman/8.0/en/replication-howto-masterstatus.html)