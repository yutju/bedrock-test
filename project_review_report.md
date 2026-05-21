#코드 리뷰 리포트

## vulnerable_app.py (167줄)

### 스타일 검사
다음은 코드에서 발견된 PEP 8 및 일반적인 스타일 규칙 위반 사항입니다.

**[라인 1-3] 위반 유형: 하드코딩된 민감 정보 (Hardcoded Secrets)**
API 키, DB 비밀번호, 사용자명 등 민감한 정보가 소스 코드에 직접 하드코딩되어 있습니다. 환경 변수나 별도의 설정 파일을 사용해야 합니다.

**[라인 26] 위반 유형: import 위치 오류 (Import Not at Top of File)**
`import hashlib`가 함수 정의들 사이에 위치해 있습니다. PEP 8에 따르면 모든 import 문은 파일 최상단(모듈 docstring 및 주석 다음)에 위치해야 합니다. `import os` 및 `import pickle`도 사용되고 있으나 import 문 자체가 누락되어 있습니다.

**[라인 32-33, 36-37, 39-40 등] 위반 유형: 함수 간 빈 줄 누락 (Missing Blank Lines Between Functions)**
PEP 8에 따르면 최상위 함수 및 클래스 정의 사이에는 2개의 빈 줄이 있어야 합니다. `same_early`, `in_treatment`, `during_yeah`, `step_forward`, `trade_push`, `door_least` 등 여러 함수들 사이에 빈 줄이 1개이거나 없습니다.

**[라인 43] 위반 유형: 내장 함수명 재사용 (Shadowing Built-in Name)**
`type = 59`는 Python 내장 함수 `type`을 변수명으로 사용하고 있어 권장되지 않습니다.

**[라인 67] 위반 유형: 내장 함수명 재사용 (Shadowing Built-in Name)**
`help = 65`는 Python 내장 함수 `help`를 변수명으로 사용하고 있어 권장되지 않습니다.

**[라인 전반] 위반 유형: 사용되지 않는 변수 (Unused Variables)**
각 함수 내에서 선언된 변수(`throw`, `bar`, `far` 등)가 실제로 사용되지 않고, 대신 정의되지 않은 `rand_var_name`을 출력하고 있습니다. 이는 정의되지 않은 변수 참조(NameError) 및 불필요한 변수 선언 문제입니다.

**[라인 전반] 위반 유형: 대문자 변수명 (Non-conventional Variable Names)**
`I = 74`, `Republican = 67`처럼 대문자로 시작하는 변수명은 PEP 8에서 클래스명에 사용하는 명명 규칙(CapWords)과 혼동될 수 있으며, 일반 변수는 소문자 snake_case를 사용해야 합니다.

**[라인 전반] 위반 유형: 함수의 의미 없는 내용 (Meaningless Function Bodies)**
수많은 함수들이 의미 없는 변수 할당과 정의되지 않은 변수(`rand_var_name`) 출력만 수행하고 있어 코드 품질 및 유지보수성이 매우 낮습니다.

### 보안 검사
코드에서 발견된 보안 취약점 목록입니다:

---

## 🔴 1. 하드코딩된 민감 정보 (Critical)

**위치:** 파일 상단 전역 변수
```python
SECRET_API_KEY = 'eef42a468f55c5c5bd0de9eaf397673f'
DB_PASSWORD = 'aL(BE1hGL('
USERNAME = 'qmoran'
```
- **유형:** Hardcoded Credentials / Secrets
- **심각도:** 🔴 Critical
- **문제:** API 키, DB 비밀번호, 사용자명이 소스코드에 평문으로 노출됨. 코드가 GitHub 등에 올라갈 경우 즉시 탈취 가능.
- **수정 제안:** 환경변수 또는 별도 시크릿 관리 도구(예: `.env` 파일 + `python-dotenv`, AWS Secrets Manager 등) 사용
```python
import os
SECRET_API_KEY = os.environ.get('SECRET_API_KEY')
DB_PASSWORD = os.environ.get('DB_PASSWORD')
```

---

## 🔴 2. SQL Injection (Critical)

**위치:** `get_user_data()` 함수
```python
query = f"SELECT * FROM users WHERE id = '{user_id}'"
```
- **유형:** SQL Injection (OWASP A03:2021 – Injection)
- **심각도:** 🔴 Critical
- **문제:** 사용자 입력(`user_id`)이 검증 없이 SQL 쿼리에 직접 삽입됨. `' OR '1'='1` 같은 입력으로 전체 DB 탈취 가능.
- **수정 제안:** Parameterized Query(매개변수화된 쿼리) 사용
```python
query = "SELECT * FROM users WHERE id = %s"
cursor.execute(query, (user_id,))
```

---

## 🔴 3. Command Injection (Critical)

**위치:** `run_command()` 함수
```python
os.system(f"echo Running command: {cmd}")
```
- **유형:** OS Command Injection (OWASP A03:2021 – Injection)
- **심각도:** 🔴 Critical
- **문제:** 사용자 입력이 OS 명령어에 직접 삽입되어 임의 명령 실행 가능. 예: `cmd = "; rm -rf /"` 입력 시 시스템 파괴 가능.
- **수정 제안:** `subprocess` 모듈을 리스트 형태로 사용하고 입력값 검증
```python
import subprocess
subprocess.run(["echo", f"Running command: {cmd}"], check=True)
```

---

## 🔴 4. XSS (Cross-Site Scripting) (High)

**위치:** `render_html_unsafe()` 함수
```python
html = f"<div>Hello, {user_input}</div>"
```
- **유형:** Reflected XSS (OWASP A03:2021 – Injection)
- **심각도:** 🔴 High
- **문제:** 사용자 입력이 HTML에 그대로 삽입됨. `<script>alert('XSS')</script>` 같은 입력으로 악성 스크립트 실행 가능.
- **수정 제안:** HTML 이스케이프 처리
```python
import html
safe_input = html.escape(user_input)
output = f"<div>Hello, {safe_input}</div>"
```

---

## 🔴 5. 안전하지 않은 역직렬화 (High)

**위치:** `load_user_unsafe()` 함수
```python
return pickle.loads(serialized_data)
```
- **유형:** Insecure Deserialization (OWASP A08:2021 – Software and Data Integrity Failures)
- **심각도:** 🔴 High
- **문제:** `pickle`은 신뢰할 수 없는 데이터를 역직렬화할 경우 임의 코드 실행 가능.
- **수정 제안:** 신뢰할 수 있는 데이터만 역직렬화하거나, `JSON` 등 안전한 형식 사용
```python
import json
def load_user_safe(serialized_data):
    return json.loads(serialized_data)
```

---

## 🟡 6. 취약한 해시 알고리즘 (Medium)

**위치:** `hash_password_weak()` 함수
```python
return hashlib.md5(password.encode()).hexdigest()
```
- **유형:** Weak Cryptographic Algorithm (OWASP A02:2021 – Cryptographic Failures)
- **심각도:** 🟡 Medium
- **문제:** MD5는 충돌 취약점이 있으며, 레인보우 테이블 공격에 매우 취약함. 비밀번호 해싱에 부적합.
- **수정 제안:** `bcrypt`, `argon2`, 또는 `hashlib.pbkdf2_hmac` 등 비밀번호 전용 해시 사용
```python
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```

---

## 📋 취약점 요약표

| # | 위치 | 유형 | 심각도 |
|---|------|------|--------|
| 1 | 전역 변수 | 하드코딩된 민감 정보 | 🔴 Critical |
| 2 | `get_user_data()` | SQL Injection | 🔴 Critical |
| 3 | `run_command()` | Command Injection | 🔴 Critical |
| 4 | `render_html_unsafe()` | XSS | 🔴 High |
| 5 | `load_user_unsafe()` | 안전하지 않은 역직렬화 | 🔴 High |
| 6 | `hash_password_weak()` | 취약한 해시(MD5) | 🟡 Medium |

---

