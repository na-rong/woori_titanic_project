# 🚢타이타닉 데이터 시각화 미니 프로젝트🚢

---

> 📄 프로젝트 명 : MySQL + ELK 구축 및 titanic 데이터 분석 및 시각화

> 👥 팀원 : [박지원](https://github.com/jiione) [최나영](https://github.com/na-rong) [곽병찬](https://github.com/gato-46) [최수연](https://github.com/lotuxsoo)

> 📆 날짜 : 2024-07-26
> 

---

- 목차
  - MySQL + ELK 구축
  - 트러블슈팅
  - 수정
  - 주제 선정

---

# 1. ELK 실행

- Elasticsearch 실행

```bash
cd elasticsearch-7.11.2
./bin/elasticsearch
```

- 키바나 실행

```java
username@servername:~$ cd kibana-7.11.2-linux-x86_64/
username@servername:~/kibana-7.11.2-linux-x86_64$ ./bin/kibana
```

---

# 2. DB-connector 다운로드 (MYSQL)

`mysql-connector-j_9.0.0-1ubuntu24.04_all.deb` 패키지를 설치하면 MySQL JDBC 드라이버가 시스템에 설치됩니다. 이 DEB 패키지는 MySQL JDBC Connector/J를 Ubuntu 24.04 시스템에 쉽게 설치할 수 있게 해줍니다.

### 1. **DEB 패키지 다운로드**

먼저, DEB 패키지를 다운로드합니다. 다운로드 링크를 클릭하거나 wget를 사용하여 다운로드할 수 있습니다:

```bash
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j_9.0.0-1ubuntu24.04_all.deb
```

### 2. **패키지 설치**

다운로드한 `.deb` 패키지를 `dpkg` 명령어를 사용하여 설치합니다:

```bash
sudo dpkg -i mysql-connector-j_9.0.0-1ubuntu24.04_all.deb
```

설치 후, 필요한 경우 `apt`를 사용하여 누락된 종속성을 해결합니다:

```bash

sudo apt-get install -f
```

### 3. **드라이버 파일 위치 확인**

패키지가 설치되면 JDBC 드라이버 `.jar` 파일은 일반적으로 `/usr/share/java/` 또는 `/usr/share/mysql/` 디렉터리에 위치합니다. 드라이버 파일이 설치된 위치를 확인합니다:

```bash
ls /usr/share/java
ls /usr/share/mysql
```

 mysql-connctor-java-<version>.jar라는 이름으로 저장됨.

`logstash-plugin` 명령어는 Logstash의 설치 디렉토리에서 실행해야 합니다. 일반적으로 Logstash는 `/usr/share/logstash` 또는 `/opt/logstash`와 같은 디렉토리에 설치됩니다.

- connector 경로

```bash
"/usr/share/java/mysql-connector-java-9.0.0.jar" 
```

# 3. Logstash input plugin 설치

Logstash의 설치 경로를 확인한 후, 해당 디렉토리로 이동하여 `logstash-plugin` 명령어를 실행해야 합니다. 일반적으로 Logstash가 `/usr/share/logstash`에 설치되어 있다면, 다음과 같이 명령어를 실행할 수 있습니다.

```bash
./bin/logstash-plugin install logstash-input-jdbc
```

![image](https://github.com/user-attachments/assets/e24e26bf-9f5a-4a9f-abb0-1f29d6250bf8)

이미 설정 파일이 있다고 뜸 

오류 메시지에 따르면, `logstash-input-jdbc` 플러그인이 이미 `logstash-integration-jdbc` 패키지에 포함되어 있습니다. 이 경우, 별도로 `logstash-input-jdbc` 플러그인을 설치할 필요가 없습니다. 대신 이미 설치된 플러그인을 사용하여 MySQL과 ELK 스택을 동기화할 수 있습니다.

### 1. **Logstash 설정 파일 수정**

Logstash 설정 파일에서 JDBC 드라이버 라이브러리 경로를 올바르게 설정합니다. 드라이버 파일의 위치를 확인한 후, Logstash 설정 파일을 다음과 같이 수정합니다:

```bash
input {
  jdbc {
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_driver_library => "/usr/share/java/mysql-connector-java-9.0.0.jar"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/fisa"
    jdbc_user => "root"
    jdbc_password => "root"
    statement => "SELECT * FROM titanic_raw"
    jdbc_paging_enabled => true
    jdbc_page_size => 1000
    schedule => "*/5 * * * * *"
    use_column_value => true
    tracking_column => "passengerid"
    last_run_metadata_path => "C:\02.devEnv\ELK\data\titanic_raw"
  }
}
filter {
  fingerprint {
    source => "passengerid"
    target => "[@metadata][fingerprint]"
    method => "SHA256"
  }
}
output {
  # 콘솔창에 어떤 데이터들로 필터링 되었는지 확인
  stdout {
    codec => rubydebug
  }
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "titanic-%{+YYYY.MM.dd}"
    document_id => "%{[@metadata][fingerprint]}"
    #action => "create"
    action => "update"
    doc_as_upsert => tr
  }
} 
```

### 2. **Logstash 재시작**

설정을 완료한 후, Logstash를 재시작하여 변경 사항을 적용합니다:

```bash
sudo systemctl restart logstash
sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/mysql_logstash.conf
```

### 3. 참고

- index 삭제

```bash
curl -X DELETE "localhost:9200/your_index_name"
```

---

# ⚠️ 트러블슈팅

- conf 파일 오류
    - 위치 경로 파악해서 잘 넣기
        - jdbc_driver_library => "/usr/share/java/mysql-connector-java-9.0.0.jar"

```
input {
  jdbc {
    jdbc_driver_library => "/usr/share/java/mysql-connector-java-9.0.0.jar"  # 실제 >
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"  # 최신 드라이버 클래스
    jdbc_connection_string => "jdbc:mysql://localhost:3306/fisa"
    jdbc_user => "root"
    jdbc_password => "root"
    statement => "SELECT * FROM titanic_raw"

  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "titanic_index"
  }
}
```

- config파일의 filter 코드 수정

```bash
# filter 수정 전 코드
filter {
  fingerprint {
    source => ["pclass", "sex", "age", "sibsp", "parch", "fare", "embarked", "survival", "ticket", "cabin"]
    target => "[@metadata][fingerprint]"
    method => "SHA1"
  }
}

#filter 수정 후 코드
filter {
  fingerprint {
    source => "passengerid"
    target => "[@metadata][fingerprint]"
    method => "SHA256"
  }
 }

```

# 🔧 수정

- ‘source’ → ‘passengerid’
    - 단일 필드인 ‘passengerid’를 사용하여 고유한 지문( fingerprint) 생성
    - passengerid가 각 데이터 항목의 고유 식별자 역할을 하므로, 고유한 문서 ID를 생성한다.
- ‘method’ → ‘SHA256’
    - SHA256해시 알고리즘 사용하여 지문 생성
    - SHA1보다 강력한 보안성 제공

- config 파일의 output 코드 수정

```bash
#output 수정 전 코드
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "titanic_index"
  }
}
 
#output 수정 후 코드 
output {
  # 콘솔창에 어떤 데이터들로 필터링 되었는지 확인
  stdout {
    codec => rubydebug
  }
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "titanic-%{+YYYY.MM.dd}"
    document_id => "%{[@metadata][fingerprint]}"
    #action => "create"
    action => "update"
    doc_as_upsert => tr
  }
} 
```

# 🔧 수정(코드 추가)

```bash
 #Elasticsearch에 문서를 삽입할 때 사용할 문서 ID를
 #filter 섹션의 fingerprint 플러그인이 생성한 고유 식별자를 사용한다. 
 document_id => "%{[@metadata][fingerprint]}"
 #Elasticsearch에서 문서가 존재할 경우 해당 문서를 업데이트하도록 지시한다.
 action => "update"
 #위에 코드와 함께 사용되며, 업데이트할 문서가 존재하지 않을 경우 문서를 삽입(upsert)하도록 지시한다. 
 doc_as_upsert => tr
```

- 즉, 세 가지 설정을 통해 동일한 데이터는 항상 같은 문서 ID를 갖게 된다.
- 문서가 존재하면 업데이트하고, 없으면 새로 삽입하여 데이터의 중복을 방지하고 최산 상태를 유지한다.

---

# 📄 주제 선정

- 아이디어
    - 탑승 항구별 평균 가격과 가격별 생존 확률, 가족 여부와 탑승 항구
    - 요금과 나이의 상관관계와 생존율
    - 생존자/사망자 분포와 요금 지불 차이
    - 요금/나이/가족관계 /생존율
    - 가족 여부에 따른 생존
    - 클래스 별 가족 여부에 따른 생존
    - 요소에 따른 생존율과 원인 분석

- 주제 선정 - 요소에 따른 생존율과 원인 분석

---
![image](https://github.com/user-attachments/assets/2e4621e3-e088-4c06-9406-9a32ef0c4103)
![image](https://github.com/user-attachments/assets/d70e8b8f-762e-4757-a1df-a32f7d03ccea)
![image](https://github.com/user-attachments/assets/9ace5a3c-b858-4f0e-bbd8-6d20e1291d29)
---

> 인사이트
> 
- 1등석 승객이 생존율이 높은것을 보아 1등석이 구조에 더 유리한 위치에 있었을 가능성이 높다. 또한 고급 서비스와 시설을 이용한 승객이 구조 과정에서 우선권을 받았을 수 도 있다.
- 가족과 함께 여행한 승객의 혼자 여행한 승객보다 생존율이 높다. 이를 보아 구조중에 가족 구성원이 서로 돕는 경향이 있을 수도 있어 생존율이 증가했을 수 있다는 도출이 나온다.
- 총 인원에서는 남성이 많았지만 생존율은 여성이 월등히 높은 것을 볼 수 있다.  이 결과는 당시에 "Women and children first" 라는 규칙이 잘 지켜졌음을 반영.
- 또한 나이대를 보면 어린아이와 고령자가 20~30대보다 생존율이 높은 것을 볼 수 있는데 이는 어린아이와 여성, 고령 나이대를 우선적으로 구조하려는 노력이 있었음을 볼 수 있다.

---

# 💡 고찰

- 최나영 : csv파일이 아닌 mysql에 있는 데이터를 연동하여 시각화 실습을 해보니 더 많은 데이터를 관리하거나 시각화 할 때 유용하다고 생각했고, config파일 설정을 꼼꼼하게 하여 데이터가 중복화되지 않도록 하는것이 중요하다는 것을 깨닫는 시간이였습니다.
- 박지원 : 실제 데이터베이스와 연동하여 ELK 파이프라인을 구축한 경험이 의미가 깊었습니다. 또한, 오랜만에 등원하여 파이프라인을 구축하는 데 어려움이 많았는데 도와준 분들께 감사했습니다.
- 곽병찬 : MySQL과 ELK를 연동하면서 데이터가 연동되는 과정과 데이터가 이동하는 경로를 한번 더 생각해보면서 환경을 구축하는 과정이 수업을 복습하는 과정에서 큰 도움이 되었다고 생각합니다. 또한 conf파일 설정을 통해서 데이터가 중첩되는 것을 해결하는 과정을 통해 성장했다고 생각합니다.
