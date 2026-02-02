# [Troubleshooting] Kafka SASL/SCRAM 인증 및 외부 리스너 설정 이슈 해결

Docker 환경에서 KRaft 모드로 구축된 Kafka 브로커에 SASL/SCRAM 보안 인증을 적용하며 발생한 접속 타임아웃 및 인증 실패 이슈를 계층별로 분석하고 해결한 기록입니다.

보안상 포트는 0000 으로 기록합니다.

---

## 1. 개요 (Context)
- **환경**: Docker (KRaft 모드), Confluent `cp-kafka:7.5.0`
- **상황**: 외부 포트(0000) 접속 시 타임아웃 발생 및 SASL 인증 정보 불일치로 인한 권한 오류 발생.

---

## 2. 주요 문제 원인 및 해결 방법

### ✅ [이슈 1] 외부 접속 타임아웃 (Advertised Listeners 설정)

* **원인**: 브로커의 `KAFKA_ADVERTISED_LISTENERS` 설정이 실제 외부 클라이언트가 도달할 수 있는 호스트/포트와 불일치하여 발생.

* **해결**: 클라이언트 환경에 맞춰 `localhost:0000`를 정확히 매핑하고, 내부(Broker-to-Broker) 리스너와 외부(Client-to-Broker) 리스너의 통신 경로를 분리함.

### ✅ [이슈 2] JAAS 설정 파일 (`.conf`)의 정적 유저 형식 오류

* **원인**: `KafkaServer` 블록 내 유저 정의 시 `user_` 접두사(Prefix)를 누락하여 브로커가 해당 계정을 유효한 유저로 인식하지 못함.

* **비교**:
  - **As-Is (오류)**: `admin="admin1!";`
  - **To-Be (수정)**: `user_admin="admin1!";`

* **주의**: `KAFKA_OPTS` 환경변수를 통해 로드되는 파일이므로, 수정 후 반드시 **컨테이너를 재시작**해야 반영됨.

### ✅ [이슈 3] Java AdminClient 유저 생성 로직의 멱등성 문제

* **원인**: 기존 유저 존재 여부를 체크(`existingUsers.contains`)하여, 계정은 존재하나 비밀번호가 변경된 경우 `alter` 로직이 실행되지 않는 현상 발생.

* **해결**: 조건문을 주석 처리하거나 업데이트 로직을 강제 수행하여, 설정값이 바뀔 때마다 브로커 메타데이터(`alterUserScramCredentials`)에 즉시 반영되도록 수정함.

### ✅ [이슈 4] Shell 특수문자(`!`) 해석 오류

* **원인**: 비밀번호에 포함된 느낌표(`!`)가 리눅스 쉘의 '히스토리 확장' 기능으로 인식되어 문자열이 깨진 상태로 전달됨.

* **해결**: `set +H` 명령으로 쉘 기능을 일시적으로 끄거나, 명령어를 **작은따옴표(`'`)**로 감싸서 특수문자가 변조되지 않도록 처리함.

---

## 3. 최종 검증 명령어 (Cheat Sheet)

### ✅ 브로커 내 유저 정보 수동 갱신 (Java 코드 오류 배제용)

```bash
docker exec -it aon-kafka kafka-configs --bootstrap-server localhost:0000 \
--alter --add-config 'SCRAM-SHA-256=[password=heily1!]' \
--entity-type users --entity-name heily
```

### ✅ ACL 설정 확인 (Read/Write 권한)

```Bash
docker exec -it aon-kafka kafka-acls --bootstrap-server localhost:0000 \
--command-config /etc/kafka/kafka_server_jaas.conf \
--list --topic response
```

### ✅ 최종 메시지 전송 테스트

```Bash
docker exec -it aon-kafka kafka-console-producer --bootstrap-server localhost:0000 \
--topic response \
--producer-property security.protocol=SASL_PLAINTEXT \
--producer-property sasl.mechanism=SCRAM-SHA-256 \
--producer-property sasl.jaas.config='org.apache.kafka.common.security.scram.ScramLoginModule required username="heily" password="heily1!";'
```

```Bash
echo "test request message" | docker exec -i aon-kafka kafka-console-producer \
--bootstrap-server localhost:0000 \
--topic request \
--producer-property security.protocol=SASL_PLAINTEXT \
--producer-property sasl.mechanism=SCRAM-SHA-256 \
--producer-property sasl.jaas.config='org.apache.kafka.common.security.scram.ScramLoginModule required username="heily" password="heily1!";'
```
---

## 4. 핵심 요약 및 회고

정적 유저 규격: jaas.conf 설정 시 유저 아이디 앞에 user_ 접두사는 필수임.

동적 설정의 우선순위: AdminClient나 kafka-configs를 통해 생성된 정보는 브로커 메타데이터에 저장되며 최신 상태를 유지하므로, 항상 명령어로 최종 확인이 필요함.

인프라와 쉘 환경: 보안 설정 시에는 애플리케이션 로직뿐만 아니라 런타임 환경(Shell, Docker Network)에 대한 이해가 수반되어야 함을 배움.