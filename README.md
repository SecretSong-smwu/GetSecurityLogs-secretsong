# GetSecurityLogs-secretsong

AI 에이전트가 "최근 보안 로그 보여줘" 같은 자연어 요청을 처리할 때 호출하는 조회 Lambda입니다. 데이터 수집 파이프라인이 DynamoDB에 저장한 원본 로그를 조회·필터링·정렬해서 반환합니다.

## 개발 배경 (v2로 재작성한 이유)

처음에는 S3에 저장된 `raw/auth.log` 파일을 직접 파싱하는 방식으로 구현했습니다. 그러나 데이터 파이프라인 담당자의 로그 수집 Lambda가 원본 로그를 **DynamoDB 테이블에 직접 저장**하는 방식으로 변경되면서, 팀이 합의한 데이터 계약이 바뀌었습니다. 이에 맞춰 S3 파싱 로직을 폐기하고 DynamoDB 조회 방식으로 전면 재작성했습니다.

## 동작 방식

1. 이벤트에서 조회 조건(`limit`, `src_ip`, `priority`)을 파싱합니다.
2. DynamoDB 테이블을 `Scan`하며 조건에 맞는 `FilterExpression`을 적용합니다.
3. 결과가 많을 경우 `LastEvaluatedKey`로 이어서 스캔합니다(최대 1000건).
4. `timestamp` 기준 최신순으로 정렬 후 `limit` 개수만큼 잘라서 반환합니다.

## 입력 (Event)

```json
{
  "queryStringParameters": {
    "limit": "20",
    "src_ip": "192.168.1.100",
    "priority": "HIGH"
  }
}
```
모든 파라미터는 선택값이며, 없으면 `limit=20`으로 최신 로그를 반환합니다.

## 출력

```json
{
  "statusCode": 200,
  "count": 3,
  "logs": [
    {
      "log_id": "2026-08-13T05:20:00+00:00_ab12cd",
      "src_ip": "192.168.1.100",
      "signature": "SSH Invalid User Scan",
      "priority": "MEDIUM",
      "action": "DENY",
      "timestamp": "2026-08-13T05:20:00+00:00",
      "description": "존재하지 않는 계정 'admin'으로 192.168.1.100에서 SSH 로그인 시도"
    }
  ]
}
```

## 환경 변수

| 변수 | 설명 |
|---|---|
| `LOG_TABLE` | 원본 로그가 저장된 DynamoDB 테이블 이름 (데이터 수집 Lambda와 동일한 테이블) |

## 필요 IAM 권한

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:Scan", "dynamodb:Query", "dynamodb:GetItem"],
  "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/<LOG_TABLE>"
}
```

## 데이터 계약

| 필드 | 타입 | 설명 |
|---|---|---|
| `log_id` | string | 고유 식별자 |
| `src_ip` | string | 출발지 IP |
| `signature` | string | 탐지 시그니처명 |
| `priority` | string | `HIGH` \| `MEDIUM` \| `LOW` |
| `action` | string | `ALLOW` \| `DENY` |
| `timestamp` | string (ISO8601) | 발생 시각 |
| `description` | string | 사람이 읽을 수 있는 설명 |

## 관련 리소스

- 데이터 소스: [A1Producer-secret](../A1Producer-secret) 이 저장하는 DynamoDB 테이블
- 호출자: AgentCore Gateway가 등록한 AI 에이전트 Tool

## 기술 스택

Python 3.12 · boto3 · Amazon DynamoDB · AWS Lambda
