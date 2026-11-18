# REST·WebSocket 연동 명세

이 문서는 v1.1 설계를 기준으로 Vision·백엔드·프론트엔드가 주고받을 메시지를 설명합니다. 아래 값은 **한 좌석만 있는 가상 실습실**의 예시입니다. 세 파트에서 같은 방(`IT4-101`)과 좌석(`A-01`)을 따라가면 메시지 사이의 관계를 확인할 수 있습니다. [초기 설계 v1.1](https://github.com/KNUCreaterCrew/vision/blob/main/zeroframe_v1.1_local_multimodal.md)

예시들은 서버가 막 시작된 방에서 이어지는 하나의 흐름입니다. 10:15:30에 A1이 사용 중으로 바뀌고, 10:15:40에 새 관측 없는 하트비트가 오고, 10:16:11에 장치 연결이 끊긴 것으로 처리되고, 10:17:05에 장치가 다시 보고합니다. 양쪽 테스트는 이 순서와 값을 그대로 입력과 기대값으로 쓸 수 있습니다.

## 1. Vision이 좌석 상태를 보고합니다

### 장치가 지금 만드는 값

2026-11-17에 확인한 Vision의 `dev`에는 백엔드 송신기와 로컬 개발용 수신기가 있습니다. 송신기는 좌석 ID별 `SeatState`를 아래 요청 형식으로 바꿔 보내고, 수신기(`mock_backend.py`)는 같은 형식을 검증합니다. `main`에는 아직 송신기가 없습니다. [Vision 송신기](https://github.com/KNUCreaterCrew/vision/blob/a201b5eceb418dd717f62b14828d00d6212ba52d/zeroframe/transport.py) · [개발 브랜치 상태 모델](https://github.com/KNUCreaterCrew/vision/blob/a201b5eceb418dd717f62b14828d00d6212ba52d/zeroframe/state.py)

| 장치 안의 값 | 백엔드 연동 시 의미 |
| --- | --- |
| 좌석 ID와 `status` | 좌석별 보고의 `seat_id`, `status`가 됩니다. 상태 코드는 `OCCUPIED`, `AWAY`, `EMPTY`, `UNKNOWN`입니다. 실카메라 경로에서 `EMPTY` 확정에 필요한 검증기는 아직 연결되지 않았습니다. |
| `status_since`, `last_observed_at` | 내부에서는 경과 시간을 재는 단조 시계의 초 단위 값입니다. 송신기는 실행을 시작할 때의 실제 시각(UTC)을 기준으로 이 값을 ISO 8601 시각으로 바꿉니다. 요청에는 `last_observed_at`만 싣고, 상태 변경 시각(`status_since`)은 백엔드가 정합니다. |
| `reason` | 가림, 낮은 신뢰도, 관측 만료 등 장치가 알아낸 사유입니다. 송신기는 마지막 관측이 30초를 넘은 좌석을 `UNKNOWN`·`STALE_OBSERVATION`으로 바꿔 보냅니다. `EDGE_OFFLINE`은 보고가 끊긴 뒤 백엔드가 판단합니다. |

### 연동할 요청 형식

장치는 **맡은 좌석 전체**를 한 보고에 담습니다. 좌석의 상태나 사유가 바뀌면 즉시 보내고, 바뀌지 않아도 10초마다 보냅니다. 이 주기적인 보고를 하트비트라고 부릅니다. 보고는 한 번에 하나만 보내며, 실패하면 지난 보고를 다시 보내지 않고 최신 상태로 새 보고를 만들어 다시 시도합니다. 원본 영상, 좌석 이미지, 객체 탐지 상자와 카메라 좌표는 요청에 포함하지 않습니다. 아래는 10시 15분 29초에 사용 중임을 확인한 좌석을 가정한 **테스트용 요청**입니다.

```http
POST /api/v1/rooms/IT4-101/seats/status
Content-Type: application/json
X-Edge-Api-Key: <장치 인증 키>
```

```json
{
  "room_id": "IT4-101",
  "timestamp": "2026-11-24T10:15:30+09:00",
  "inference_fps": 2.1,
  "saved_frames_count": 0,
  "seats": [
    {
      "seat_id": "A-01",
      "status": "OCCUPIED",
      "confidence": null,
      "decision_source": "YOLO",
      "last_observed_at": "2026-11-24T10:15:29+09:00"
    }
  ]
}
```

| 필드 | 읽는 방법 |
| --- | --- |
| `timestamp` / `last_observed_at` | 장치의 보고 시각 / 좌석의 마지막 유효 관측 시각입니다. 시각은 시간대가 포함된 ISO 8601 형식으로 주고받습니다. 장치는 UTC(`+00:00`)로 보내고, 이 문서의 예시는 읽기 쉽도록 한국 시간으로 적습니다. |
| `confidence` | 검증 가능한 신뢰값이 있으면 0~1, 없으면 `null`입니다. |
| `decision_source` | 선택 필드입니다. 판정 경로를 `YOLO`, `YOLO_VLM`, `FALLBACK` 중 하나로 나타냅니다. |
| `last_observed_at`, `reason` | 선택 필드입니다. 유효 관측이 없으면 관측 시각을 생략할 수 있습니다. 확인 불가 원인은 아래 코드로 표현합니다. |
| `inference_fps` / `saved_frames_count` | YOLO의 실제 분석 FPS / 장치가 보고하는 저장 프레임 수입니다. |

확인 불가 사유는 가림(`OCCLUDED`), 낮은 신뢰도(`LOW_CONFIDENCE`), 모델 간 불일치(`MODEL_CONFLICT`), 검증 시간 초과(`VLM_TIMEOUT`), 관측 만료(`STALE_OBSERVATION`), 장치 통신 두절(`EDGE_OFFLINE`)입니다.

**수신 성공 응답 · HTTP 200**

```json
{
  "ok": true,
  "updated_count": 1,
  "server_time": "2026-11-24T10:15:30.112+09:00"
}
```

### 백엔드가 보고를 반영하는 규칙

인증과 형식 검사를 모두 통과한 보고만 반영합니다. 하나라도 실패하면 방 상태는 그대로입니다.

| 응답 | 경우 |
| --- | --- |
| 200 | 반영했습니다. 바뀐 좌석이 없어도 200입니다. |
| 401 | `X-Edge-Api-Key`가 없거나 틀렸습니다. |
| 404 | 등록되지 않은 방입니다. |
| 422 | 형식이 틀렸거나, 등록되지 않은 좌석이 섞였거나, 경로와 본문의 `room_id`가 다릅니다. |

통과한 보고는 좌석마다 다음처럼 반영합니다.

1. **상태와 사유**는 장치가 보낸 `status`·`reason`으로 바꿉니다. 저장된 값과 `status`나 `reason`이 달라진 좌석을 **변경된 좌석**이라고 부르며, 이 좌석만 `SEAT_UPDATED`로 알립니다.
2. **마지막 확인 시각**은 `last_observed_at`이 있으면 그 값으로 바꾸고, 없으면 기존 값을 유지합니다.
3. **상태 변경 시각**(`status_since`)은 `status`가 바뀐 좌석만 새로 정합니다. `OCCUPIED`·`AWAY`·`EMPTY`가 되면 그 좌석의 `last_observed_at`을, `UNKNOWN`이 되면 보고의 `timestamp`를 씁니다.
4. **보고에 없는 좌석**은 바꾸지 않습니다. 방에 등록돼 있어도 장치가 맡지 않은 좌석이 여기에 해당합니다.
5. **`updated_count`**는 변경된 좌석 수입니다.

같은 보고가 다시 도착해도 변경된 좌석이 없으므로 이벤트와 집계가 중복되지 않습니다. 장치가 보고를 하나씩 보내므로 백엔드는 장치의 `timestamp`로 보고를 버리지 않습니다. 장치를 다시 켠 직후에는 시각 동기화 전이라 장치 시계가 뒤로 가 있을 수 있기 때문입니다. 드물게 늦게 처리된 보고가 있어도 10초 안에 도착하는 다음 보고가 바로잡습니다.

| 시각 | 기준 시계 | 쓰임 |
| --- | --- | --- |
| `timestamp` | 장치 | 보고를 만든 시각입니다. `UNKNOWN`으로 바뀐 시각을 정할 때 씁니다. |
| `last_observed_at` | 장치 | 좌석을 마지막으로 유효하게 관측한 시각입니다. 학생 화면의 마지막 확인 시각입니다. |
| `server_time` | 서버 | 서버가 보고를 받은 시각입니다. 30초 연결 판단은 이 시각으로만 합니다. |
| `status_since` | 백엔드가 정함 | 상태가 바뀐 시각입니다. 위 3번 규칙을 따르고, 장치 끊김 처리 때는 서버가 전환한 시각, 서버 시작 직후에는 서버 시작 시각입니다. |

서버를 막 시작한 방은 `edge_online`이 `false`이고 모든 좌석이 `UNKNOWN`·`EDGE_OFFLINE`입니다. 아직 보고를 받지 못한 것도 장치 통신 두절로 봅니다. `last_observed_at`은 `null`입니다. 첫 보고를 받으면 장치 연결과 좌석 상태가 함께 바뀝니다. 위 예시 요청이 그 첫 보고이며, A1이 `UNKNOWN`에서 `OCCUPIED`로 바뀌었으므로 `updated_count`는 1입니다.

### 새 관측이 없는 하트비트

10초 뒤인 10:15:40의 하트비트입니다. A1의 상태와 마지막 관측 시각이 첫 보고와 같습니다.

```json
{
  "room_id": "IT4-101",
  "timestamp": "2026-11-24T10:15:40+09:00",
  "inference_fps": 2.0,
  "saved_frames_count": 0,
  "seats": [
    {
      "seat_id": "A-01",
      "status": "OCCUPIED",
      "confidence": null,
      "decision_source": "YOLO",
      "last_observed_at": "2026-11-24T10:15:29+09:00"
    }
  ]
}
```

```json
{
  "ok": true,
  "updated_count": 0,
  "server_time": "2026-11-24T10:15:40.105+09:00"
}
```

변경된 좌석이 없으므로 이벤트를 보내지 않습니다. 방 조회 결과도 2번 예시와 같고, A1의 마지막 확인 시각은 10:15:29로 남습니다. 서버는 이 보고를 받은 시각을 장치 연결 판단의 기준으로 기록합니다.

## 2. 웹이 방 전체 상태를 읽습니다

`GET /api/v1/rooms/IT4-101`은 좌석 배치와 최신 상태, 방 집계를 함께 반환합니다. 예시 방에는 사용 중인 좌석 한 개가 있으므로 빈자리와 확인 불가 좌석이 모두 0개입니다.

```json
{
  "room_id": "IT4-101",
  "room_name": "제1실습실",
  "total_seats": 1,
  "available_seats": 0,
  "unknown_seats": 0,
  "congestion_level": "HIGH",
  "edge_online": true,
  "version": 1,
  "seats": [
    {
      "seat_id": "A-01",
      "label": "A1",
      "status": "OCCUPIED",
      "reason": null,
      "status_since": "2026-11-24T10:15:29+09:00",
      "last_observed_at": "2026-11-24T10:15:29+09:00",
      "has_outlet": true,
      "is_window": true,
      "layout": { "x": 120, "y": 80, "width": 64, "height": 48 }
    }
  ]
}
```

`available_seats`는 `EMPTY`, `unknown_seats`는 `UNKNOWN`인 좌석 수입니다. v1.1의 혼잡도 기준은 전체 좌석 중 빈자리 비율이 60% 이상이면 `LOW`, 25% 이상 60% 미만이면 `MODERATE`, 25% 미만이면 `HIGH`입니다. 화면에는 확인 불가 좌석 수도 함께 표시합니다. 좌석의 `reason`은 `UNKNOWN`인 좌석의 사유이고, 그 밖의 상태에서는 `null`입니다. `version`은 3번에서 설명합니다.

## 3. 웹이 이후 변경을 받습니다

웹은 `/ws/rooms/IT4-101`에 WebSocket으로 연결합니다. 다음 예시는 10:15:40 하트비트를 마지막으로 장치가 30초 이상 응답하지 않은 경우입니다.

**장치 연결 변경 · `EDGE_STATUS_CHANGED`**

```json
{
  "event": "EDGE_STATUS_CHANGED",
  "room_id": "IT4-101",
  "version": 2,
  "edge_online": false
}
```

**좌석 변경 · `SEAT_UPDATED`**

```json
{
  "event": "SEAT_UPDATED",
  "room_id": "IT4-101",
  "version": 2,
  "available_seats": 0,
  "unknown_seats": 1,
  "congestion_level": "HIGH",
  "changes": [
    {
      "seat_id": "A-01",
      "prev_status": "OCCUPIED",
      "curr_status": "UNKNOWN",
      "reason": "EDGE_OFFLINE",
      "status_since": "2026-11-24T10:16:11+09:00",
      "last_observed_at": "2026-11-24T10:15:29+09:00"
    }
  ]
}
```

`changes`에는 `status`나 `reason`이 바뀐 좌석만 담기며, 각 항목은 바뀐 뒤 좌석의 상태·사유·상태 변경 시각·마지막 확인 시각을 담습니다. 웹은 이 값으로 해당 좌석의 표시를 고치고, 집계 숫자는 이벤트의 값으로 바꿉니다. 한 번도 관측하지 못한 좌석의 `last_observed_at`은 `null`입니다.

장치가 끊기면 백엔드는 해당 방 좌석을 모두 확인 불가로 바꿉니다. 웹은 좌석 변경 이벤트를 반영하거나 방 전체를 다시 조회해 같은 집계를 표시할 수 있습니다. 이때 A1의 상태 변경 시각은 10:16:11, 마지막 확인 시각은 여전히 10:15:29입니다. 오프라인이 이어지는 동안 같은 이벤트를 다시 보내지 않습니다.

### 장치가 다시 연결되면

10:17:05에 장치가 다시 보고합니다. 장치는 끊긴 동안에도 관측을 이어 갔고, 10:17:04에 A1에서 사람을 확인했습니다.

```json
{
  "room_id": "IT4-101",
  "timestamp": "2026-11-24T10:17:05+09:00",
  "inference_fps": 2.1,
  "saved_frames_count": 0,
  "seats": [
    {
      "seat_id": "A-01",
      "status": "OCCUPIED",
      "confidence": null,
      "decision_source": "YOLO",
      "last_observed_at": "2026-11-24T10:17:04+09:00"
    }
  ]
}
```

```json
{
  "ok": true,
  "updated_count": 1,
  "server_time": "2026-11-24T10:17:05.087+09:00"
}
```

백엔드는 `version` 3으로 `EDGE_STATUS_CHANGED`(`edge_online: true`)와 `SEAT_UPDATED`를 보냅니다. `SEAT_UPDATED`의 A1 변경은 `UNKNOWN`에서 `OCCUPIED`, 사유 `null`, 상태 변경 시각과 마지막 확인 시각 모두 10:17:04입니다. 재연결만으로 이전 상태를 되살리지 않고 보고에 담긴 상태를 반영합니다. 장치는 30초 넘게 관측하지 못한 좌석을 `UNKNOWN`·`STALE_OBSERVATION`으로 보내므로, 오래된 빈자리가 다시 표시되지 않습니다.

### 변경을 놓치지 않는 순서

`version`은 방 상태(좌석, 집계, 장치 연결)가 바뀔 때마다 1씩 커지는 번호입니다. 한 번의 변경으로 함께 나가는 이벤트는 같은 번호를 씁니다. 위 흐름에서는 첫 보고 반영 후 1, 장치 끊김 처리 후 2, 재연결 후 3입니다.

1. 웹은 WebSocket을 먼저 연결합니다.
2. 방 전체를 조회하고 응답의 `version`을 기억합니다. 조회 응답 전에 도착한 이벤트는 모아 둡니다.
3. 기억한 `version` 이하인 이벤트는 버리고 나머지를 차례로 반영합니다.

WebSocket이 끊기면 1번부터 다시 합니다. 서버를 다시 시작하면 번호가 0부터 시작하지만, 재접속할 때마다 방 전체를 다시 조회하므로 기준 번호도 함께 바뀝니다.

### 마지막 확인 시각을 맞추는 방법

웹은 WebSocket을 연결해 둔 동안에도 **30초마다 방 전체를 다시 조회**합니다. 조회 결과로 화면을 바꾸고, 응답의 `version`을 새 기준으로 삼아 위 3번 규칙을 그대로 적용합니다.

WebSocket은 서버가 보낸 것만 웹에 도착합니다. 서버는 `status`나 `reason`이 바뀐 좌석만 `SEAT_UPDATED`로 알리므로, 상태가 그대로이고 관측 시각만 새로워지는 하트비트는 알리지 않습니다. 다시 조회하지 않으면 마지막 확인 시각이 화면을 연 때에 머뭅니다.

| 시각 | 일어난 일 | 서버의 A1 마지막 확인 | 다시 조회하지 않는 화면 |
| --- | --- | --- | --- |
| 14:00:00 | 학생이 A1에 앉아 `SEAT_UPDATED`가 나감 | 14:00:00 | 14:00:00 |
| 14:00:10 ~ 14:30:00 | 10초마다 하트비트, 상태는 사용 중 그대로 | 14:30:00 | 14:00:00 |

30분 동안 학생은 실제로는 10초 전에 확인한 정보를 30분 전 정보로 읽게 됩니다. 30초마다 다시 조회하면 화면의 시각은 서버보다 최대 30초 늦습니다. 다시 조회하는 과정에서 혹시 놓친 이벤트도 함께 복구됩니다.

하트비트마다 관측 시각만 담은 이벤트를 따로 보내는 방법도 검토했습니다. 시각은 가장 정확하지만, 상태가 그대로여도 10초마다 방의 모든 웹에 메시지가 나가고 이벤트 종류가 하나 늘어납니다. 마지막 확인 시각이 10초 늦은지 30초 늦은지는 학생이 자리를 고르는 데 큰 차이가 없다고 보고 주기적 조회를 택했습니다. 상태가 바뀌는 순간의 시각은 `SEAT_UPDATED`의 `last_observed_at`으로 바로 맞춥니다.

### 서버가 연결을 닫는 경우

| 종료 코드 | 경우 | 웹이 할 일 |
| --- | --- | --- |
| `4404` | 등록되지 않은 방으로 연결했습니다. | 다시 연결하지 않고 방이 없다고 표시합니다. |
| `1013` | 서버가 보낸 이벤트를 1초 안에 받지 못했습니다. 느린 연결 하나가 같은 방의 다른 웹과 장치 보고를 붙잡지 않도록 서버가 끊습니다. | 다시 연결하고 방 전체를 조회합니다. |

그 밖의 이유로 끊겨도 웹은 다시 연결하고 방 전체를 조회합니다. 놓친 이벤트를 하나씩 다시 받지 않고, 조회 결과로 화면을 맞춘 뒤 이후 이벤트부터 반영합니다.
