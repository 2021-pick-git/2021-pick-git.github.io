---
layout: post
title: DB 리플리케이션 적용기
date: 2021-09-30 13:32:20 +0300
description: DB replication으로 DB 서버를 다중화한 과정을 기록한다.
img: database.png
tags: [백엔드, Database]
---

## INTRO
- DB Replication을 MySQL 공식 홈페이지에서 찾아보면 다음과 같이 말한다.    
  > Replication enables data from one MySQL databse server (known as a source) to be copied to one or more MySQL database servers (know as replicas) 
  출처 : [링크](https://dev.mysql.com/doc/refman/8.0/en/replication.html)

- 즉, 하나의 데이터베이스(master/source)에서 다른 하나 또는 그 이상의 데이터베이스(slaves/replicas)로 데이터를 복제하여 저장하는 것이다. 
- Replication은 비동기로 동작한다. 따라서 replicas가 master에 지속적으로 연결되어는 동기식으로 동작하지 않는다. 
- 설정에 따라서 여러 데이터베이스, 선택된 데이터베이스, 선택된 테이블에만 replication을 적용할 수도 있다. 

<br>
<br>

## MySQL replication 장점
공식 홈페이지에 나와있는 장점 4가지는 다음과 같다. 
1. Scale-out solutions
   - 다수의 replicas를 두고 load를 분산해서 퍼포먼스를 높이는 장점이 있다. 
   - 대부분의 replication 적용이유이기도 하다. 
   - 쓰기 및 업데이트는 master 서버에서 이루어진다. 
   - 조회는 하나 또는 여러 slave 서버에 분산되서 처리된다. 
2. Data security
   - master 서버와 slave 서버가 분리되어 있으므로 하나의 slave 서버에 문제가 생겨도 다른 slave 서버에 영향을 미치지 않고 데이터를 보존할 수 있다. 
   - 하지만 Master server에 장애가 생기면 문제가 생긴다. 
3. Analytics
   - 실시간 데이터 생성 및 업데이터가 master 서버에서 이루어지는 동안 데이터 분석처리는 slave 서버에서 처리하여 master 서버에 성능저하를 전혀 일으키지 않도록 지원한다. 
4. Long-distance data distribution
   - 리모트에 필요한 데이터를 위한 local 데이터 복제를 master에 접촉하지 않고 slave 서버에서 처리할 수 있다. 

더 많은 정보를 위해서는 다음 [링크](https://dev.mysql.com/doc/refman/8.0/en/replication.html)를 참고한다. 

<br>
<br>   

## Replication 적용하기 

### 적용이유 
- 현재 진행중인 [프로젝트](pick-git.com)에 DB replication을 적용하기로 했다. 그 이유는 프로젝트가 SNS의 일종이므로 유저에 의한 페이지 이동이 잦고 그것에 따른 조회 쿼리가 매우 많기 때문이다. 따라서 Master server 1개, slave server 2개를 두어 조회 쿼리를 slave 서버 2개로 분산했다. 

### 참고사항 
- 현재 본 프로젝트의 WAS가 AWS EC2 인스턴스에서 실행 중이며 이번에 DB Replication을 적용하면서 DB 서버를 분리했다. 
- AWS EC2 인스턴스 3개를 추가로 생성해서 MySQL master 서버 1개 + slave 서버 2개를 구성했다. 
- 쓰기 및 업데이트 작업은 master, 조회는 2개의 slave 서버를 RR(Round Robin) 방식으로 분산처리하도록 구성했다. DB는 MariaDB를 사용한다. 
- 조회 작업은 Transaction의 read-only 속성을 통해 확인하고 slave db를 연결했다. 

<br>

### 적용하기 
DB replication 적용에는 크게 3가지 단계가 있다. 다음 [레포지토리](https://github.com/2021-pick-git/db-replication-learning-test)에 가면 적용을 위한 replication 학습테스트 코드를 확인할 수 있다. 

1. Remote 서버에 MariaDB 로컬 설치 및 기본 설정
2. Master 서버와 Slave 서버 replication 연결 설정
3. 프로덕션 코드에 DB 수동 연결 및 (여러 slave 서버를 두고 있다면) slave DB 선택 로직 구현 

<br>

#### 1-1) MariaDB 설치 및 기본 설정 
- 우분투에 MariaDB를 설치한다.  
  ```shell
  $ sudo apt update
  $ sudo apt install mariadb-server
  ```
+ 현재 프로젝트를 위해서 제공받은 AWS 권한은 많이 닫혀있으므로 사용 가능한 포트(9000)으로 바꾸어주었다. [포트변경방법](https://bskyvision.com/1049)

- 프로젝트에 사용할 database를 생성한다. 
  - 현재 우리 프로젝트에서 사용하는 database는 `pickgit`이다.

- 각 DB 서버에 계정을 생성한다. 
  ```mysql
  create user 'replication'@'%' identified by 'password';
  ```
  - 계정 이름 뒤에 %로 지정해야 전체에서 접속이 허용된다.

<br>

#### 2-1) Master DB 설정 
- 해당 계정에 권한을 부여한다. (master)
  ```mysql
  $ grant all privileges on {database}.* to 'replication'@'%'; 

  $ flush privileges;
  ```
- 위와 같이 하면 해당 계정에 대한 전체 권한이 열린다. 불안하다면 다음과 같이 replication에 대한 권한만 설정해도 된다. 
    ```mysql
    $ grant replication slave on *.* to 'replication'@'%'; 

    $ flush privileges;
    ```
  - 참고로 replication slave 권한을 줄 때는 `*.*`로 주지 않으면 db grant 및 global privileges 경고가 뜬다. 

- 설정과정
    <p align="center"><img width="600" alt="masterDb" src="https://user-images.githubusercontent.com/63405904/132974481-47521392-72e8-4596-9c08-0481b572717a.png"></p>


- 다음 경로의 설정파일을 열어 수정한다. master db 서버의 서버 id를 설정하는 과정이다. 
    - 설정파일 경로 
    <p align="center"><img width= 600 src="https://user-images.githubusercontent.com/63405904/132974223-9df3b0cb-68fc-451f-bcb5-703980a36a79.png"></p>

    - 설정 수정 
    <p align="center"><img width="600" alt="masterDb" src="https://user-images.githubusercontent.com/63405904/132974298-64ba1690-5e69-441d-ba8e-b7ccc8887c43.png"></p>

- 모든 설정이 끝난 뒤에 mysql를 재실행하여 설정을 적용한다. 
    ```shell
    $ sudo service mysqld restart
    ```

- Master DB 정보를 다음 명령어로 확인한다. 
    - File 값과 position 값으로 slave db에 master db에 대한 정보를 설정해야 한다. 
        ```mysql
        $ show master status;
        ```

    <p align="center"><img width="600" alt="스크린샷 2021-09-05 오후 4 25 23" src="https://user-images.githubusercontent.com/63405904/132974579-e455d052-e082-40f3-b68d-858692cbbb79.png"></p>

  - 위 두 정보가 의미하는 것이 무엇인지 확인하고 싶다면 다음 [링크](https://dev.mysql.com/doc/refman/5.7/en/replication-howto-masterstatus.html)를 참고하자. 
  
    > The File column shows the name of the log file and the Position column shows the position within the file. In this example, the binary log file is mysql-bin.000003 and the position is 73. Record these values. You need them later when you are setting up the replica. They represent the replication coordinates at which the replica should begin processing new updates from the source. 
    - 간단히 말하면 replica가 master db의 데이터를 읽을 binary 파일과 읽기 시작할 위치인 position에 대한 정보이다. 

<br>

#### 2-2) Slave DB 설정 
- Master DB 과 동일하게 다음 설정경로로 가서 `server-id`를 수정한다. 
    - 현재 master의 `server-id`가 1이므로 `slave1`은 2, `slave2`는 3으로 설정해주었다. 
    ```shell
    $ sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
    ```

- Slave Db에서 이전에 기록해둔 Master DB의 정보를 입력해 두 DB를 연결한다. 
  ```mysql
  mysql> change master to master_host={master_db_ip}, master_port={master_db_port}, master_user={master_username}, master_password={master_password}, master_log_file={master_bin_file}, master_log_pos={position};
  ```

- Slave DB를 실행시킨다. 
  ```mysql
  mysql> start slave;
  ```

- 실행 시키고 다음 명령어를 치면 slave db의 상태와 master와의 연결상태 여부를 확인할 수 있다. 
  
    <p align="center"><img width="600" alt="스크린샷 2021-09-05 오후 4 25 23" src="https://user-images.githubusercontent.com/63405904/132976950-0bb84655-da17-4b4b-8d44-c03626b46e51.png"></p>

- 이제 master db에 데이터를 추가하면 slave db에도 적용이 되는 것을 확인할 수 있다. 

<br>

#### 3-1) Springboot DB configuration 설정 - datasource 정보 기입
- DB 서버에서 하는 설정은 Master DB에 쓰기 및 업데이트 처리시 Slave DB에 적용이 되도록 하는 연결 설정이다. 
- 이외의 datasource를 선택하고, 설정에 맞게 connection을 만들고, 실제 쿼리를 처리하도록 하는 것은 어플리케이션 코드에서 구현을 해야한다. 
  <p align="center"><img width=600 src="https://user-images.githubusercontent.com/63405904/132996798-4b1a4c22-484c-415b-acdb-d599cdb0fcc2.png"><br>이전 yml datasource 설정</p>
  <br>

- 다음과 같이 datasource 정보를 `yml` 혹은 `properties`에 기록한다. 
    <p align="center"><img width=600 src="https://user-images.githubusercontent.com/63405904/132996912-59fcf5f4-2cd9-40c7-89e0-9a142c5d20f8.png"><br>datasource 정보</p>

<br>

- 위 `yml에` 기입한 `datasource` 정보를 활용하기 위해서 다음과 같은 객체를 만들어 `yml` 정보를 바인딩 한다. 
  - 유의할 점은 내부에 선언된 정보를 위해서는 `static inner class`를 칼럼과 동일한 이름으로 생성해야 한다. 그러면 class 내부의 자료구조로 정보가 들어간다.
  - `getter` 및 `setter`가 필수적으로 있어야한다. 
    ```java
    @ConfigurationProperties(prefix = "datasource") //이 annotation을 활용해서 yml 정보를 매핑한다.
    public class MasterDataSourceProperties {

        private final Map<String, Slave> slave = new HashMap<>();

        private String url;
        private String username;
        private String password;

        //getter 및 setter 
        //slave map에 대한 setter는 불필요하다.

        public static class Slave { //중첩 데이터 명과 일치해야한다. 즉, datasource.slave의 두번째 요소와 동일한 이름으로 static class를 만들어야한다.

            private String name;
            private String url;
            private String username;
            private String password;

            //getter 및 setter 생략 
        }
    }
    ```
<br>

#### 3-2) Springboot DB configuration 구현
- 위 입력한 datasource는 하나가 아니기 때문에 자동으로 연결이 안되고 상황에 따라 다른 datasource가 연결이 된다. 해당 작업을 수동으로 해야하기 때문에 몇가지 직접 설정해야하는 것들이 있다.
   
- 1) 첫번째는 적합한 상황에 다른 datasource를 제공하는 설정이다. 하나 이상의 datasource를 생성해 저장한다. 

- 2) 두번째는 Jpa에 대한 entityManagerFactory `@bean` 설정이다. 본래 datasource가 자동연결되면서 JPA에 대한 설정도 되지만 여기서는 수동으로 해야한다. 
  - 이때 datasource가 매번 바뀌므로 entityManagerFactory 생성시 `LazyConnectionDataSourceProxy` 로 프록시 datasource를 연결해준다. 

- 3) 세번재는 TransactionManager에 대한 설정이다. 이 또한 수동으로 datasource를 관리하려고 하니 추가해야하는 부분이다. 

- 4) 기존에 자동으로 Datasource를 연결하던 설정을 해제하고, 수동으로 연결할 datasource의 properties를 지정해주어야 한다. (class 상단에 annotation으로 설정)

```java
@Configuration
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class) //4)
@EnableConfigurationProperties(MasterDataSourceProperties.class) //4)
public class DataSourceConfiguration {

    private final MasterDataSourceProperties dataSourceProperties;
    private final JpaProperties jpaProperties;

    public DataSourceConfiguration(
        MasterDataSourceProperties dataSourceProperties,
        JpaProperties jpaProperties
    ) {
        this.dataSourceProperties = dataSourceProperties;
        this.jpaProperties = jpaProperties;
    }

    @Bean //1)번 부분
    public DataSource routingDataSource() {
        DataSource master = createDataSource(
            dataSourceProperties.getUrl(),
            dataSourceProperties.getUsername(),
            dataSourceProperties.getPassword()
        );

        Map<Object, Object> dataSources = new HashMap<>();
        dataSources.put("master", master);
        dataSourceProperties.getSlave().forEach((key, value) ->
            dataSources.put(value.getName(), createDataSource(
                value.getUrl(), value.getUsername(), value.getPassword()
            ))
        );

        ReplicationRoutingDataSource replicationRoutingDataSource = new ReplicationRoutingDataSource();
        replicationRoutingDataSource.setDefaultTargetDataSource(dataSources.get("master"));
        replicationRoutingDataSource.setTargetDataSources(dataSources);

        return replicationRoutingDataSource;
    }

    private DataSource createDataSource(String url, String username, String password) {
        return DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .url(url)
            .driverClassName("org.mariadb.jdbc.Driver")
            .username(username)
            .password(password)
            .build();
    }

    @Bean //2)번 부분
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        EntityManagerFactoryBuilder entityManagerFactoryBuilder =
            createEntityManagerFactoryBuilder(jpaProperties);

        return entityManagerFactoryBuilder.dataSource(dataSource())
            .packages("com.pickgit.dbreplicationlearningtest")
            .build();
    }

    private EntityManagerFactoryBuilder createEntityManagerFactoryBuilder(
        JpaProperties jpaProperties
    ) {
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        return new EntityManagerFactoryBuilder(vendorAdapter, jpaProperties.getProperties(), null);
    }

    private DataSource dataSource() {
        return new LazyConnectionDataSourceProxy(routingDataSource());
    }

    @Bean //3)번 부분
    public PlatformTransactionManager transactionManager(
        EntityManagerFactory entityManagerFactory
    ) {
        JpaTransactionManager jpaTransactionManager = new JpaTransactionManager();
        jpaTransactionManager.setEntityManagerFactory(entityManagerFactory);
        return jpaTransactionManager;
    }
}
```
<br>

### 3-3) 조회 쿼리시 datasource를 RR으로 선택하는 로직
- 현재 연결가능한 datasources들을 순회하면서 쓰기 및 업데이트면 master, 조회시에는 slave를 번갈아 선택하는 로직을 구현한다. 
  ```java
  public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {

        private static final Logger LOGGER = LoggerFactory.getLogger(ReplicationRoutingDataSource.class);

        private SlaveNames slaveNames;

        @Override
        public void setTargetDataSources(Map<Object, Object> targetDataSources) {
            super.setTargetDataSources(targetDataSources);

            List<String> replicas = targetDataSources.keySet().stream()
                .map(Object::toString)
                .filter(string -> string.contains("slave"))
                .collect(toList());

            this.slaveNames = new SlaveNames(replicas);
        }  

        @Override
        protected String determineCurrentLookupKey() {
            boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly(); //조회 쿼리인 경우 
            if (isReadOnly) {
                String slaveName = slaveNames.getNextName(); //다음 slave 선택 

                LOGGER.info("Slave DB name: {}", slaveName);

                return slaveName;
            }

            return "master";
        }
    }

    public class SlaveNames {

        private final String[] value;
        private int counter = 0;

        public SlaveNames(List<String> slaveDataSourceProperties) {
            this(slaveDataSourceProperties.toArray(String[]::new));
        }

        public SlaveNames(String[] value) {
            this.value = value;
        }

        public String getNextName() {
            int index = counter;
            counter = (counter + 1) % value.length;
            return value[index];
        }
    }
  ```

<br>
<br>

## Replication Test 하기
- Member를 입력하고 조회를 여러번 했을 때 의도된 대로 replication이 적용되는지 확인한다. 
- `@DataJpaTest`로도 진행할 수 있으나, 빈으로 등록된 설정 요소들이 필요하기 때문에 `@SpringBootTest`로 테스트를 진행했다. (@`DataJpaTest`를 진행하면서 해당 configuration만 빈으로 등록하는 방식으로 테스트해도 무방하다.) 
- datasource를 자동 연결하지 않는 설정 annotation을 class 상단에 추가해야한다. 
  
```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = Replace.NONE) //datasource 자동연결 x
@ActiveProfiles("db")
class MemberRepositoryTest {

    @Autowired
    private MemberRepository memberRepository;

    @DisplayName("Master DB에 데이터를 추가하면 slave DB에도 반영된다.")
    @Test
    void addMember_Success() {
        // given
        Member member = new Member("pickgit", 29);
        memberRepository.save(member);
    }

    @DisplayName("Slave DB에서 데이터를 조회한다 - 여러번 조회시 slave db를 번갈아 조회한다.")
    @Test
    void findMember_Success() {
        // given
        Member member = memberRepository.save( new Member("pickgit", 29));

        // when
        Member findMember1 = memberRepository.findById(member.getId())
            .orElseThrow();
        Member findMember2 = memberRepository.findById(member.getId())
            .orElseThrow();
        Member findMember3 = memberRepository.findById(member.getId())
            .orElseThrow();
        Member findMember4 = memberRepository.findById(member.getId())
            .orElseThrow(); //조회 할 때마다 사용 DB 로거가 번갈아서 찍힌다. 

        // then
        assertThat(findMember1.getId()).isNotNull();
        assertThat(findMember1.getName()).isEqualTo(member.getName());
        assertThat(findMember1.getAge()).isEqualTo(member.getAge());
    }
}
```

- 결과 화면: 4번의 조회를 할때 1, 2 slave DB가 번갈아 선택된다. 
  <p align="center"><img width=600 src="https://user-images.githubusercontent.com/63405904/133017838-c022df09-9d81-407e-9eac-7cd1e5af5de6.png"></p>

<br>
<br>

## 마주한 이슈
- yml에 properties 입력시 오타 주의 !! (자동완성 안해주기 때문에 접미사 -s 등을 주의해야함)
- JPA 정보 또한 자동연결할 때 해주는 설정들을 하나씩 다 명시해주어야한다. 
- 기존에 ddl 전략을 외부에 기입했다면 왜인지 `hbm2ddl.auto=create`로 지정해야 적용이 되었다. 
  - 현재 JPA properties 
    ```yml
    spring:
    jpa:
        properties:
        hibernate:
            show-sql: true
            dialect: org.hibernate.dialect.MySQL8Dialect
            format_sql: true
            physical_naming_strategy: org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy
            hbm2ddl:
                auto: create
        generate-ddl: true
    ```

<br>

- Slave DB 권한부여 및 bind address 오픈
    - Host의 접근이 허가되지 않는다는 오류가 날 때는 다음 두가지를 해주어야한다. 
      1. slave db 계정 생성 및 권한 부여 (위 master db에 했던 작업과 동일)
      2. `sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf`에 bind-address 부분 `0.0.0.0` 으로 지정

    <p align="center"><img width=600 src="https://user-images.githubusercontent.com/63405904/133018579-e3f0221f-a767-4203-9b07-18bc41fc7f7c.png"></p>
    
    - 다음 [링크](https://blog.naver.com/6116949/221991858055)를 참고하자.

<br>

- Springboot JPA에 대한 Hibernate Naming Strategy 지정
  - Springboot에서 자동으로 지정할 때는 알아서 네이밍전략이 설정되었으나, 수동을 할 때는 이 부분도 yml에 기입해주어야 한다. 
  - 그렇지 않으면 테이블 및 칼럼명이 그대로 camel case로 입력된다. 
  - yml에 다음 설정을 해서 DB에서 underscore로 지정되도록 전략을 지정한다. 
    `physical_naming_strategy: org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy`

<br>
<br>

## 번외) 기존 DB의 데이터 dump 하기 
- 기존 DB에 있던 데이터들을 새로 생성한 master db에 옮기기 위해 `mySqldump`를 사용해 마이그레이션을 했다. 
    ```shell
    $ mysqldump -u [사용자 계정] -p [원본 데이터베이스명] > [생성할 백업 파일명].sql #백업 sql 생성 

    #scp를 사용해 새로운 database가 있는 서버로 sql 파일 이동

    $ mysql -u [사용자 계정] -p [복원할 DB] < [백업된 DB].sql #sql 파일을 사용해 데이터 복원 
    ```

- Master DB에만 적용하면 slave DB에 알아서 적용이 된다. 


<br>
<br>

**[참고자료]**
- [https://mycup.tistory.com/237](https://mycup.tistory.com/237)
- [https://velog.io/@max9106/DB-Spring-Replication](https://velog.io/@max9106/DB-Spring-Replication)
- [https://mangkyu.tistory.com/97](https://mangkyu.tistory.com/97)
- [https://velog.io/@kyujonglee/Mysql-%EB%B0%B1%EC%97%85-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98](https://velog.io/@kyujonglee/Mysql-%EB%B0%B1%EC%97%85-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98)

<br>
<br>

> 백엔드 코다입니다 🙌