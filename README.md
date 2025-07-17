# 🔍 ELK 스택을 이용한 소비 패턴 분석 프로젝트

로그 시각화 실습 및 ELK 학습을 위한 프로젝트입니다. Ubuntu 가상 머신에 ELK 스택(Elasticsearch, Logstash, Kibana)을 설치하여 카드 데이터에 대한 시각화를 진행했습니다.

> ✅ Filebeat → Logstash → Elasticsearch → Kibana 파이프라인을 구축
> 
> 
> ✅ VirtualBox 기반 내부망(LAN) 접속
> 
> ✅ 보안 고려를 통해 공인 IP 포트는 열지 않음
> 

---

## 🧱 시스템 아키텍처

<img src="https://file.notion.so/f/f/9d9842c3-5bb3-4aea-84bd-5315dfa61386/ebdbd4ae-c5b6-45f0-9e03-3482d98e37c0/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7_2025-07-17_154544.png?table=block&id=2338416c-e9f3-8078-b6b4-d0c63debc00c&spaceId=9d9842c3-5bb3-4aea-84bd-5315dfa61386&expirationTimestamp=1752775200000&signature=BmEzrc7gkyR8Znq2WLglobN5GC03H9OYLipWe9HzwDo&downloadName=%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7+2025-07-17+154544.png"/>

| 구성 요소 | 설명 |
| --- | --- |
| **Elasticsearch** | 로그 데이터를 저장하고 검색 |
| **Logstash** | 수신된 로그를 필터링/정제 |
| **Kibana** | Elasticsearch 데이터를 시각화 |
| **Filebeat**  | CSV 로그 파일을 실시간으로 전송 |

```
[개인 PC (Windows)]
  ↳ edu_data_real.csv → Filebeat → Logstash (Ubuntu: 5044)

[Ubuntu 가상 머신 (VirtualBox)]
  ↳ Logstash → Elasticsearch (9200)
  ↳ Kibana (5601) 

[Windows 브라우저]
  ↳ <http://192.168.0.X:5601>

```

> ✅ VirtualBox는 "브리지 어댑터"로 설정하여 동일 네트워크에서 접속 가능하게 구성

---

## 🛠️ 설치 및 설정 절차

### 1️⃣ ELK 스택 설치 (Ubuntu)

```bash
# Java 설치
sudo apt update
sudo apt install -y openjdk-11-jdk

# Elasticsearch 설치
wget <https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.2-amd64.deb>
sudo dpkg -i elasticsearch-7.11.2-amd64.deb
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# Kibana 설치
wget <https://artifacts.elastic.co/downloads/kibana/kibana-7.11.2-amd64.deb>
sudo dpkg -i kibana-7.11.2-amd64.deb
sudo systemctl enable kibana

# Logstash 설치
wget <https://artifacts.elastic.co/downloads/logstash/logstash-7.11.2-amd64.deb>
sudo dpkg -i logstash-7.11.2-amd64.deb
sudo systemctl enable logstash

```

---

### 2️⃣ Kibana 외부 접속 설정

```bash
sudo nano /etc/kibana/kibana.yml
```

외부에서 kibana에 접속할 수 있도록 host 포트를 `0.0.0.0`으로 수정한다.

```yaml
server.host: "0.0.0.0"
```

---

### 3️⃣ Logstash 설정

```bash
sudo nano /etc/logstash/conf.d/churn.conf
```

적용한 설정:

```yaml
# 처리할 데이터를 받을 경로 = filebeat
input {
  beats {
    port => 5044
  }
}

filter {
	# CSV 형태의 message 필드를 분리하여 새로운 필드로 재구성
  mutate {
    split => ["message", ","]
    add_field => {
      "기준 시점"           => "%{[message][0]}"
      "고객 번호"              => "%{[message][1]}"
      ...
    }

		# 불필요한 기본 필드 제거
    remove_field => ["message", "ecs", "host", "agent", "@version", "input", "tags", "log", "@timestamp"]
  }
  
  if [SEX_CD] == "1" {
    mutate {
      update => { "SEX_CD" => "남" }
    }
  } else if [SEX_CD] == "2" {
    mutate {
      update => { "SEX_CD" => "여" }
    }
  }

  if [AGE] >= 65 {
    drop { }
  }

	if ![BAS_YH] or [BAS_YH] =~ /^.{0,5}$/ or [BAS_YH] =~ /^.{7,}$/ {
	  drop { }
	}
}

output {
  stdout {
    codec => rubydebug
  }

	# 처리한 데이터를 내보낼 경로 = elasticsearch
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "cardfisa"
  }
}

```

- 성별 칼럼의 int 값에 따라 `“남”`, `“여”` string으로 저장
- 라이프스타일 (대학생, 신혼, 은퇴자 등)에서 은퇴자의 정확성을 높이기 위해 65세 이상의 데이터 제거
    - 은퇴한 지 오래된 고령층은 은퇴자의 소비 특징을 흐릴 수 있어 분석에서 제외했습니다.

```bash
sudo systemctl start logstash
```

---

### 4️⃣ 방화벽 포트 열기 (UFW)

```bash
sudo ufw allow 5601
sudo ufw allow 9200
sudo ufw allow 5044
sudo ufw enable
```

---

### 5️⃣ 팀원 PC에서 Filebeat 설정

```yaml
# filebeat.yml (Windows)
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - C:\\mydata\\card_data.csv  # CSV 파일 경로

output.logstash:
  hosts: ["192.168.0.5:5044"]  # Ubuntu 내부 IP

```

다음 명령어를 통해 Filebeat을 실행한다.

```powershell
filebeat.exe -e -c filebeat.yml
```

---

### 6️⃣ Kibana 인덱스 패턴 등록

- 브라우저에서 접속: `http://192.168.0.5:5601`
- 좌측 메뉴 → **Discover**
- 인덱스 패턴 생성 → `card*`

---

## 📊 시각화 결과

---

## 🚀 트러블 슈팅

### 1. 🔒 외부 포트 개방 포기 → 내부망 브리지 모드로 전환

초기에는 공인 IP(`118.XXX.XXX.XXX`)를 통해 외부에서 Kibana에 접근하려 했으나, **보안상 포트를 개방하지 않기로 결정**하였습니다.

| 이유 | 설명 |
| --- | --- |
| Kibana는 기본 인증이 없음 | 외부에 노출되면 누구나 대시보드, 로그 확인 가능 |
| Elasticsearch는 API로 조작 가능 | 공격자가 데이터 조회, 삽입, 삭제 가능 |
| 보안 미구현 시 실질적 침해 가능 | 실제 사고 사례 다수 존재함 |

✅ **대안 |** 내부망(LAN) 기반 브리지 모드 사용

- VirtualBox의 네트워크 모드를 **브리지 어댑터**로 설정
- **Windows ↔ Ubuntu VM 간 내부 IP로만 통신 허용**
- 외부 인터넷에서는 절대 접근 불가 → 보안 확보

---

### 2. VirtualBox 네트워크 설정 문제 (NAT → 브리지 전환)

**문제점**

- 기본 NAT 모드에서는 외부(호스트)에서 게스트(VM) 포트(`9200`, `5601`) 접근 불가. 포트포워딩 없이는 통신이 차단됨.

**원인**

- Kibana, Elasticsearch는 외부 접근 기반 시각화가 필요하지만 NAT에서는 접속 불가.

**해결 방법**

- VirtualBox를 **브리지 어댑터**로 변경하여 **같은 네트워크 상에서 직접 접근 가능**하도록 구성 → 별도 포트포워딩 없이 통신 가능.

---

### 3. Elasticsearch 설정 누락 (elasticsearch.yml)

**문제점**

- 다음 필드가 누락됨
    
    ```yaml
    node.name: node-1
    cluster.initial_master_nodes: ["node-1"]
    ```
    

**문제 원인**

- Elasticsearch는 초기 클러스터 형성을 위해 마스터 노드 정보를 필요로 함. 위 설정 없을 경우 **단일 노드임에도 부팅 실패** 발생.

**해결 방법**

- 위 두 설정을 명시적으로 추가.

---

### 4. VM 리소스 부족 (JVM 기반 서비스 다운)

**문제점**

- 초기 VM 사양이 2 vCPU / 4GB RAM에 불과하여 Elasticsearch + Logstash를 동시에 실행 시 과부하 발생.

**문제 원인**

- JVM 기반 서비스는 메모리 소비가 크며, Garbage Collection 지연과 함께 OutOfMemory로 인한 다운 발생.

**해결 방법**

- VM 사양을 **4 vCPU / 8GB RAM**으로 상향 조정 → 서비스 안정화 성공.

---

### 5. Logstash 타입 변환 실패 (`mutate → convert`)

**문제점**

- `TOT_USE_AM`, `AGE` 등 수치 필드가 빈 문자열("") 또는 null 상태일 때, `integer` 변환 시도 → 예외 발생.

**문제 원인**

- Logstash는 빈 값에 대해 변환 실패 시 파이프라인 전체가 중단됨.
- `integer` 타입이어야 Kibana에서 `sum`, `avg` 등 수치 집계가 가능함.

**해결 방법**

- 타입 변환 전 조건문으로 값 유효성 체크 추가
    
    ```ruby
    if [TOT_USE_AM] != "" {
      mutate { convert => { "TOT_USE_AM" => "integer" } }
    }
    ```
    
---

### 6. Logstash 조건 비교 오류 (SEX_CD 변환)

**문제점**

- `if [SEX_CD] == "1"` 처럼 문자열 비교를 했지만, 실제 데이터는 숫자(1) → 조건 불일치.

**문제 원인**

- Logstash는 `"1"`(문자열)과 `1`(숫자)를 구분함 → 타입 불일치로 인해 조건문 실행되지 않음.

**해결 방법**:

- **옵션 1**: 문자열로 변환 후 비교
    
    ```ruby
    mutate { convert => { "SEX_CD" => "string" } }
    if [SEX_CD] == "1" {
      mutate { update => { "SEX_CD" => "남" } }
    }
    ```
    
- **옵션 2**: 숫자 그대로 비교
    
    ```ruby
    if [SEX_CD] == 1 {
      mutate { update => { "SEX_CD" => "남" } }
    }
    ```
    
---

## 🌕 회고 및 소감

외부 접근을 허용하려면 단순히 포트를 여는 것 이상으로 고려해야 할 설정이 많았다.

또한 실제 데이터를 시각화하면서 **타입 변환, 필드 명명, 값 유효성** 등 사소해 보이던 요소들이 분석 정확도에 큰 영향을 준다는 걸 느꼈다.
