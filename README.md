# SpeechHelpe — Backend

> 발표 중 다음 내용을 잊었을 때, **스마트워치의 이미지 힌트로 발표 흐름을 이어가도록 돕는 서비스**의 Spring Boot 백엔드 서버

<p>
  <img alt="Java 17" src="https://img.shields.io/badge/Java-17-007396?logo=openjdk&logoColor=white" />
  <img alt="Spring Boot 3.5.6" src="https://img.shields.io/badge/Spring%20Boot-3.5.6-6DB33F?logo=springboot&logoColor=white" />
  <img alt="MySQL" src="https://img.shields.io/badge/MySQL-Database-4479A1?logo=mysql&logoColor=white" />
  <img alt="STOMP WebSocket" src="https://img.shields.io/badge/STOMP-WebSocket-010101" />
  <img alt="FastAPI" src="https://img.shields.io/badge/FastAPI-SBERT-009688?logo=fastapi&logoColor=white" />
  <img alt="Wear OS" src="https://img.shields.io/badge/Wear%20OS-Image%20Hints-4285F4?logo=wearos&logoColor=white" />
</p>


---

## 📌 프로젝트 소개

발표를 준비했더라도 긴장하면 **다음 내용이 순간적으로 떠오르지 않아 흐름이 끊길 수 있습니다**.

**SpeechHelper**는 발표 음성을 실시간으로 인식하고 대본의 현재 위치를 추적해 **다음 내용을 떠올릴 이미지 힌트**를 스마트워치에 보여주는 발표 보조 서비스입니다.

> 본 레포지토리는 **프로젝트·대본 관리 · 의미 단위 분리 · 사전 임베딩 요청 · 실시간 문장 추적 결과 전달**을 담당하는 **백엔드(Spring Boot)** 서버입니다. Android·Wear OS 앱과 FastAPI 임베딩 서버는 별도 레포지토리로 운영됩니다.

**핵심 사용자 흐름**

```text
대본 등록 → 문장별 이미지 준비 → 발표 시작 → 실시간 STT → 현재 문장 추적 → 다음 이미지 워치 표시
```

---

## ✨ 주요 기능

| 기능 | 설명 |
|---|---|
| **프로젝트·대본 관리** | 발표 프로젝트와 대본 등록·조회, 문장별 이미지 URL 제공 |
| **대본 의미 단위 분리** | GPT로 대본을 의미 단위로 나누고 짧은 키워드·요약 추출 |
| **실시간 문장 추적** | 대본을 사전 임베딩하고 STOMP로 받은 발화와 의미 유사도를 비교해 현재 문장 반환 |
| **다음 힌트 연동** | 문장 전환 조건을 확인해 다음 문장 ID를 전달하고 앱이 해당 이미지를 워치에 표시 |

백엔드는 STT로 변환된 텍스트를 받아 문장 추적 결과를 반환합니다. 음성 인식·이미지 캐싱·워치 전송은 Android 앱에서 담당합니다.

---

## 🛠 기술 스택

| 구분 | 기술 |
|---|---|
| **언어 / 프레임워크** | Java 17 · Spring Boot 3.5.6 |
| **DB** | MySQL · Spring Data JPA |
| **실시간 통신** | STOMP WebSocket · SockJS |
| **대본 전처리** | OpenAI `gpt-4o-mini` |
| **임베딩** | Python FastAPI · `jhgan/ko-sroberta-multitask` |
| **클라이언트** | Android Kotlin · Google Cloud Streaming STT · Wear OS |
| **API 문서** | springdoc-openapi · Swagger UI |
| **빌드 / 배포** | Gradle · AWS EC2 · RDS |

---

## 🏗 시스템 아키텍처

<table>
  <tr>
    <td width="50%" align="center">
      <img src="https://github.com/user-attachments/assets/8d213ccf-3e2c-4c17-835d-190e9c336bc5" width="100%" alt="서버 아키텍처" />
    </td>
    <td width="50%" align="center">
      <img src="https://github.com/user-attachments/assets/dc3b93f2-d6f9-4be6-8920-57890cdab546" width="100%" alt="모델 아키텍처" />
    </td>
  </tr>
</table>



Spring Boot는 프로젝트·대본을 MySQL에 저장합니다. 현재 문장과 다음 문장은 FastAPI와 연동해 판단하고 그 결과를 발표별 STOMP 구독 경로로 전달합니다.

**스마트폰은 음성 입력·서버 통신·이미지 준비를, 워치는 힌트 표시를 담당**합니다. 발표 전에 이미지를 앱 내부 파일로 내려받고 Room에 메타데이터를 저장해 발표 중에는 네트워크 다운로드 없이 캐시에서 조회합니다.

---

## ⚙️ 핵심 동작

### 발표 준비 — 대본 전처리와 사전 임베딩

1. **대본 등록**: 대본을 문장으로 나누고 GPT로 의미 단위와 짧은 요약을 추출해 DB에 저장합니다.
2. **이미지 준비**: 문장별 힌트 이미지를 확인하고 Android 앱에서 사전 캐싱합니다.
3. **사전 임베딩**: 발표 시작 시 문장 조각을 FastAPI에 전달해 대본 임베딩을 메모리에 준비합니다. 발표 중에는 발화 텍스트를 임베딩해 대본과 비교합니다.

### 발표 진행 — 문장 추적과 이미지 힌트

```text
Google Streaming STT → STOMP 발화 전송 → SBERT 유사도 비교
→ 현재 문장·다음 문장 ID 반환 → 캐시 이미지 조회 → 워치 표시
```

전체 대본에서 가장 유사한 문장을 현재 위치로 찾고 현재 대본과 발화를 별도로 비교해 다음 문장 전환을 판단합니다. 발화가 문장 부호로 끝나고 **코사인 유사도 0.69 이상**이면 다음 문장 ID를 반환합니다. 이 값은 문장 전환 임계값입니다.

앱은 현재 문장 주변의 결과만 반영해 갑작스러운 위치 이동을 줄이고 누적 STT 결과에서 완료한 발화를 분리해 중복 전송을 줄입니다. **Streaming STT 중간 결과 활용과 이미지 사전 캐싱**을 적용한 대표 측정에서 음성 입력부터 워치 표시까지 전체 지연을 **5.110초 → 2.013초로 약 60.6% 단축**했습니다.

---

## 👥 팀

**2025-2 캡스톤디자인 — SpeechHelper / Team 07**

| 이름 | 역할 |
|---|---|
| 박시윤 | Android UI · 프로젝트/대본 API |
| 이승언 | 대본 전처리 · 이미지 캐싱 · Wear OS 연동 |
| 조영찬 | STT · WebSocket · 문장 추적 · 임베딩 서버 |

---

## 🔗 프로젝트 레포지토리

백엔드·Android/Wear OS 앱·임베딩 서버 레포지토리는 아래 그룹에서 관리합니다.

[Speech-Helper GitHub Organization](https://github.com/Speech-Helper)
