# REST·WebSocket 연동 명세

이 문서는 v1.1 설계를 기준으로 Vision·백엔드·프론트엔드가 주고받을 메시지를 설명합니다. 아래 값은 **한 좌석만 있는 가상 실습실**의 예시입니다. 세 파트에서 같은 방(`IT4-101`)과 좌석(`A-01`)을 따라가면 메시지 사이의 관계를 확인할 수 있습니다. [초기 설계 v1.1](https://github.com/KNUCreaterCrew/vision/blob/main/zeroframe_v1.1_local_multimodal.md)

## 1. Vision이 좌석 상태를 보고합니다

좌석 상태가 바뀌면 즉시 보고하고, 변화가 없어도 장치가 연결돼 있음을 알리는 보고(하트비트)를 10초마다 보냅니다. 다음은 10시 15분 29초에 처음으로 사용 중임을 확인한 좌석의 예입니다.

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
| `timestamp` / `last_observed_at` | 장치의 보고 시각 / 좌석의 마지막 유효 관측 시각입니다. 시각은 시간대가 포함된 ISO 8601 형식으로 주고받습니다. |
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
  "seats": [
    {
      "seat_id": "A-01",
      "label": "A1",
      "status": "OCCUPIED",
      "status_since": "2026-11-24T10:15:29+09:00",
      "last_observed_at": "2026-11-24T10:15:29+09:00",
      "has_outlet": true,
      "is_window": true,
      "layout": { "x": 120, "y": 80, "width": 64, "height": 48 }
    }
  ]
}
```

`available_seats`는 `EMPTY`, `unknown_seats`는 `UNKNOWN`인 좌석 수입니다. v1.1의 혼잡도 기준은 전체 좌석 중 빈자리 비율이 60% 이상이면 `LOW`, 25% 이상 60% 미만이면 `MODERATE`, 25% 미만이면 `HIGH`입니다. 화면에는 확인 불가 좌석 수도 함께 표시합니다.

## 3. 웹이 이후 변경을 받습니다

웹은 `/ws/rooms/IT4-101`에 WebSocket으로 연결합니다. 다음 예시는 1번 보고를 마지막으로 장치가 30초 이상 응답하지 않은 경우입니다.

**장치 연결 변경 · `EDGE_STATUS_CHANGED`**

```json
{
  "event": "EDGE_STATUS_CHANGED",
  "room_id": "IT4-101",
  "edge_online": false
}
```

**좌석 변경 · `SEAT_UPDATED`**

```json
{
  "event": "SEAT_UPDATED",
  "room_id": "IT4-101",
  "available_seats": 0,
  "unknown_seats": 1,
  "congestion_level": "HIGH",
  "changes": [
    {
      "seat_id": "A-01",
      "prev_status": "OCCUPIED",
      "curr_status": "UNKNOWN",
      "reason": "EDGE_OFFLINE",
      "status_since": "2026-11-24T10:16:01+09:00"
    }
  ]
}
```

장치가 끊기면 백엔드는 해당 방 좌석을 모두 확인 불가로 바꿉니다. 웹은 좌석 변경 이벤트를 반영하거나 방 전체를 다시 조회해 같은 집계를 표시할 수 있습니다. 이때 A1의 상태 변경 시각은 10:16:01, 마지막 유효 관측은 여전히 10:15:29입니다. 장치가 돌아온 뒤에는 새로 확인된 좌석부터 상태를 갱신합니다.

## 구현 전에 맞춰 볼 두 가지

- **보고에 포함할 좌석 범위.** 상태 변경 보고와 하트비트에 전체 좌석을 보낼지, 변경된 좌석만 보낼지 정해야 합니다. 수신 측이 생략된 좌석의 상태를 어떻게 다룰지도 여기에 달려 있습니다.
- **상태가 같을 때의 마지막 확인 시각.** 같은 자리를 계속 관측하면 마지막 확인 시각은 바뀌지만 좌석 변경 이벤트는 발생하지 않습니다. 웹에 최신 시각을 보여주는 방법과 재연결 직후 전체 상태를 다시 읽는 시점을 연동할 때 정합니다.
