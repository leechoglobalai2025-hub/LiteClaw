# 🐾 LiteClaw — 보안 최우선 AI 중앙 제어 시스템

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-NVIDIA%20DGX%20Spark-76B900.svg)](https://www.nvidia.com/)
[![Paper](https://img.shields.io/badge/논문-AI시대의_오픈소스-orange.svg)](https://leechoglobalai.com/ai-시대의-오픈소스-설계-사상·도면·sop-흐름도의-공개/)

> 🇰🇷 한국어 | 🇨🇳 [中文](README.zh.md) | 🇺🇸 [English](README.md)

**안전한 AI 어시스턴트 — 제로 트러스트 아키텍처**

> OpenClaw의 "삽질"에서 자체 개발까지, LiteClaw는 한 사람, 한 대의 머신, 그리고 분노 속에서 탄생한 AI Agent 중앙 제어 시스템입니다.

---

## 📖 탄생 이야기

### 제1장: OpenClaw과의 만남 (2026년 1월)

2026년 1월 31일, 당시 가장 핫했던 오픈소스 AI Agent 플랫폼 **OpenClaw**(GitHub Stars 145,000+)을 설치했습니다. 정말 마음에 들었습니다——5일 후 모든 것이 무너지기 시작하기 전까지는.

### 제2장: API 요금의 악몽

처음 **Claude API**를 연결했습니다. 처음에는 정상적으로 보였습니다——대화당 $0.03이었지만, 비용이 지속적으로 $0.10까지 상승했습니다. 더 충격적인 것은: **input Token이 output Token의 50배**였습니다. 이것은 정상적인 비율이 절대 아닙니다.

그래서 더 저렴한 **Gemini API**로 전환했습니다——Google이 제공한 $300 무료 크레딧이 있었기 때문에 사실상 무료였습니다.

### 제3장: TPM 제한으로 차단

Gemini API 연결 4일째, 문제가 폭발했습니다:

- Telegram 지속적 오류 발생
- OpenClaw 완전 사용 불가
- Google 백엔드 확인 결과, **Gemini API의 TPM(분당 토큰 수) 제한**에 걸림

![Gemini API 백엔드 데이터](images/gemini-api-rate-limit.jpg)
*Gemini API 백엔드: Gemini 3 Pro의 TPM이 1.26M/1M에 도달, 제한 초과*

GPT에게 물어본 결과 놀라운 답변을 받았습니다: **분당 1M Token 사용량은 개인 사용자에게는 거의 불가능한 수준이며, 기업이 스트레스 테스트를 할 때만 발생하는 상황입니다!**

하루 동안 차단되었고, 한국 시간 오후 4시에 해제되었습니다.

### 제4장: 근본 원인 분석

차단 해제 후 첫 번째로 한 일은 대화 압축 플러그인을 설치한 것이었습니다. 하지만 이것도 하루밖에 버티지 못했습니다——**다시 제한에 걸렸습니다**.

그래서 OpenClaw의 아키텍처를 깊이 분석한 결과, 근본적인 문제를 발견했습니다:

> **OpenClaw은 모든 대화 기록을 지속적으로 쌓아올린다.**

저는 긴 컨텍스트 사용자이자 귀추적 논리 추론자로, 태생적으로 높은 Token 소비자입니다. 이는 곧: **어떤 API에 연결하든 반드시 제한에 걸리거나, 막대한 비용을 지불해야 한다는 뜻입니다.**

### 제5장: LiteClaw의 탄생 (2026년 2월 13일)

대안을 찾기 시작하여 경량 제품 **NanoBot**을 발견했습니다. 분석 후, 나만의 엔지니어링 방안을 설계했습니다.

**2월 13일**, Google의 IDE **Antigravity**에서 **Claude Opus 4.5**를 연결하여, 5시간 만에 **LiteClaw v1.0**을 완성했습니다.

![아키텍처 비교 분석](images/architecture-comparison.jpg)
*Opus 4.5가 수행한 LiteClaw vs OpenClaw vs NanoBot 아키텍처 수준 객관적 분석——사람의 분석이 아닌, AI의 분석입니다!*

| 차원 | OpenClaw | NanoBot | LiteClaw |
|------|----------|---------|----------|
| 포지셔닝 | 풀기능 AI Agent 플랫폼 | 초경량 개인 어시스턴트 | 보안 최우선 AI 중앙제어 |
| 코드량 | ~430,000 행 | ~4,000 행 | ~5,000 행 (42 파일) |
| 순환 의존성 | 존재 (대형 프로젝트 불가피) | 적음 (코드 적음) | **제로** ✅ |
| 키 보호 | ⚠️ 평문 저장 API key, session token | 기본 환경변수 | ✅ SecretValue 래퍼——str/repr/json/pickle 전체 경로 차단 |
| Agent 방화벽 | ⚠️ Docker 샌드박스 (기본 비활성화) | 없음 | ✅ Shell 8조 + 도구 4조 정규식 규칙, 기본 활성화 |
| 로그 마스킹 | ⚠️ 자동 비식별화 없음 | 없음 | ✅ 6가지 모드 자동 비식별화 |
| 감사 엔진 | 내장 security audit 명령 | 없음 | ✅ 3단계 감사 (pre/exec/post), 자동 기록 |
| Skill 보안 | ⚠️ 36% 취약, 76개 악성 Skill (Marketplace 보안 사건) | 해당 없음 | Skills 로컬 관리, Marketplace 리스크 없음 |

### 제6장: 마이그레이션과 진화

이후 LiteClaw를 **NVIDIA DGX Spark**로 이전하고, **Claude Code**로 지속적인 개발과 반복을 진행했습니다.

![LiteClaw v1 인터페이스](images/liteclaw-v1-ui.jpg)
*DGX Spark에서의 첫 번째 LiteClaw UI 인터페이스*

### 제7장: AI 프로그래밍의 돌파구와 논문 (2026년 3월)

2월 말, AI 프로그래밍 제어를 연구하던 중 갑자기 떠올랐습니다: **Claude Code의 CLI 인터페이스로 내가 설계한 플로우차트대로 프로그래밍을 완성할 수 있을까?**

**3월 28일, 세 번의 테스트:**

| 테스트 | 방법 | 결과 | 원인 |
|--------|------|------|------|
| 1차 | 직접 프로그래밍 | ❌ 실패 | 시공 플로우차트 부재 |
| 2차 | 멀티 Agent 협업 | ❌ 실패 | "레거시 코드 이관 대법" 자동 시작 |
| 3차 | 싱글 Agent 단계별 실행 | ✅ 성공 | 플로우차트대로 한 단계씩 실행 |

테스트 성공 후, 동시에 논문을 완성했습니다:

> 📄 **《AI 시대의 오픈소스 — 설계 사상·도면·SOP 흐름도의 공개》**
>
> 🇰🇷 [한국어](https://leechoglobalai.com/ai-시대의-오픈소스-설계-사상·도면·sop-흐름도의-공개/) | 🇨🇳 [中文](https://leechoglobalai.com/ai时代的开源-公开设计思路、图纸与sop流程图/) | 🇺🇸 [English](https://leechoglobalai.com/open-source-in-the-ai-era-publishing-design-blueprints-sop-flowcharts/)

**3월 31일, 다국어 AB 테스트:**

논문에서 제시한 방법론에 기반하여, 한국어·중국어·영어 세 가지 언어의 Task 파일로 AB 테스트를 진행——**전부 통과**. 어떤 언어를 사용하든 LiteClaw의 AI 프로그래밍을 완성할 수 있습니다.

![LiteClaw Desktop 현황](images/liteclaw-desktop.jpg)
*현재 버전의 LiteClaw Desktop —— 다중 LLM 프로바이더 연결 지원*

### 제8장: 현재

LiteClaw는 처음 5,000줄의 경량 도구에서, **8.3GB의 슈퍼 AI Agent 중앙 제어 시스템**으로 진화했습니다.

> 🎯 **이것은 제 논문 이론의 방법론이 올바르다는 것을 증명합니다. 이것은 또한 AI 시대 오픈소스의 새로운 장을 열었습니다——AI 프로그래밍의 95% 올바른 경로를 제어할 수 있는 Task 파일을 오픈소스합니다!**

---

## ✨ 핵심 기능

- 🔐 **제로 트러스트 보안 아키텍처** — SecretValue 래퍼, API 키 평문 표시 불가
- 🏗️ **L0-L8 8계층 아키텍처** — 엄격한 단방향 의존성, 제로 순환 의존성
- 🛡️ **3단계 감사 엔진** — pre/exec/post 자동 기록
- 🔒 **6가지 모드 자동 로그 비식별화**
- 🤖 **멀티 LLM 지원** — Google Gemini, OpenAI, Anthropic, DeepSeek, xAI, Qwen, Kimi 등
- 🖥️ **로컬 LLM 지원** — vLLM을 통한 로컬 모델 연결
- 🌐 **다국어 인터페이스** — 중국어, 영어, 한국어

---

## 🚀 사용 방법 (오픈소스 Markdown 파일)

### 우리가 오픈소스하는 것은?

**코드가 아닙니다. Python 실행이 필요하지 않습니다.** 우리가 오픈소스하는 것은 **Markdown 형식의 Task 파일**——설계 사상, 아키텍처 도면, SOP 흐름도입니다.

### 사용 방법

1. **다운로드**: 자신이 능숙한 언어 버전의 Markdown 파일 3개 다운로드 (한국어/중국어/영어)
2. **가져오기**: 아무 IDE (VS Code, Cursor 등) 또는 **Claude Code**에 가져오기
3. **연결**: Claude Opus 4.6 모델 연결
4. **단계별 실행**: Markdown의 흐름도 지시에 따라 AI가 단계별로 프로그래밍 완성

✅ **이렇게 하면 AI 프로그래밍으로 LiteClaw의 기본 아키텍처를 100% 제작할 수 있습니다.**

### 왜 이 방식이 안전한가?

> 기존 오픈소스 = 남의 코드 다운로드 → 공급망 오염, 악성코드, 백도어 등의 위험 존재

> **LiteClaw 오픈소스 = 설계 도면 다운로드 → AI가 도면대로 사용자 머신에서 제로부터 구축 → 절대적으로 안전**

- ✅ 실행 가능한 코드가 없으므로 공급망 오염 위험 없음
- ✅ 서드파티 의존성 주입 가능성 없음
- ✅ AI가 로컬 환경에서 제로부터 생성하므로 모든 코드를 검토 가능
- ✅ 자유로운 2차 개발 허용, 자유롭게 커스터마이징

> 자세한 내용은 논문을 참조하세요: [《AI 시대의 오픈소스 — 설계 사상·도면·SOP 흐름도의 공개》](https://leechoglobalai.com/ai-시대의-오픈소스-설계-사상·도면·sop-흐름도의-공개/)

---

## 🛡️ LiteClaw의 최강점: 아키텍처와 시스템 보안

LiteClaw의 가장 강력한 부분은 오늘날 "스크립트 키디"들이 가장 약한——**아키텍처 설계와 시스템 보안**입니다.

아래는 **Claude Opus 4.5의 LiteClaw에 대한 객관적 AI 분석**입니다 (인간의 주관적 평가가 아님):

| 보안 특성 | OpenClaw | NanoBot | LiteClaw |
|-----------|----------|---------|----------|
| 키 보호 | ⚠️ 평문 저장 | 기본 환경변수 | ✅ SecretValue 래퍼 |
| Agent 방화벽 | ⚠️ Docker (기본 비활성화) | 없음 | ✅ 8+4조 정규식 규칙, 기본 활성화 |
| 로그 마스킹 | ⚠️ 없음 | 없음 | ✅ 6가지 모드 자동 비식별화 |
| 감사 엔진 | 명령형 (설정 검사) | 없음 | ✅ 3단계 자동 감사 |
| Prompt 주입 방어 | ⚠️ 감사 결과 무효 | 없음 | Firewall 도구 매개변수 스캔 |
| Skill 보안 | ⚠️ 36% 취약 | 해당 없음 | ✅ 로컬 관리, Marketplace 리스크 없음 |

---

## 🛠️ 개발 타임라인

| 날짜 | 이벤트 |
|------|--------|
| 2026.01.31 | OpenClaw 설치, AI Agent 여정 시작 |
| 2026.02.05 | API 요금 이상 및 TPM 제한 문제 발생 |
| 2026.02.13 | **LiteClaw v1.0 탄생** — Antigravity + Claude Opus 4.5로 5시간 만에 완성 |
| 2026.02.14 | NVIDIA DGX Spark로 이전, Claude Code로 지속 개발 시작 |
| 2026.03.28 | CLI 싱글 Agent 프로그래밍 테스트 성공 + **논문 완성** |
| 2026.03.31 | 다국어(한/중/영) AI 프로그래밍 AB 테스트 전체 통과 |
| 2026.04.01 | LiteClaw, 8.3GB 슈퍼 AI Agent 중앙제어 시스템으로 진화 |

---

## 📄 논문

**《AI 시대의 오픈소스 — 설계 사상·도면·SOP 흐름도의 공개》**

먼저 실험 테스트를 완료한 후 논문을 작성하고, 이후 논문의 방법론으로 다국어 AB 테스트를 검증했습니다.

- 🇰🇷 [한국어 버전](https://leechoglobalai.com/ai-시대의-오픈소스-설계-사상·도면·sop-흐름도의-공개/)
- 🇨🇳 [中文版](https://leechoglobalai.com/ai时代的开源-公开设计思路、图纸与sop流程图/)
- 🇺🇸 [English Version](https://leechoglobalai.com/open-source-in-the-ai-era-publishing-design-blueprints-sop-flowcharts/)

---

## 📜 라이선스

이 프로젝트는 **Apache License 2.0** 하에 배포됩니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.

모든 다운로드 및 사용자의 자유로운 2차 개발을 허용합니다.

---

## 📬 연락처

- 🏢 **이조글로벌인공지능연구소** (LEECHO Global AI Research Lab)
- 🌐 공식 홈페이지: [https://www.leechoglobalai.com/](https://www.leechoglobalai.com/)
- 🐙 GitHub: [@leechoglobalai](https://github.com/leechoglobalai2025-hub)
- 📍 위치: 대한민국 인천
