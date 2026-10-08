# MES AI Infra

이 폴더의 `docker-compose.yml`로 플랫폼과 앱을 함께 올리고 내립니다. 앱 소스는 형제 폴더에 있어야 합니다. 예: `../backend`, `../frontend`.

| 서비스 | 이미지 | 주소 | 용도 |
| --- | --- | --- | --- |
| MariaDB 11.4 | `mariadb:11.4` | `localhost:3306` | LOT, 작업지시, 검사, 출하, 알람. DB/계정 `mes` / `mes` |
| Mosquitto 2 | `eclipse-mosquitto:2` | MQTT `1883`, WebSocket `9001` | 설비 상태, 센서, 작업 목표. 개발 중 익명 허용 |
| InfluxDB 2.7 | `influxdb:2.7` | `localhost:8086` | 진동 파형과 특징값. org `mes`, bucket `sensors`, token `mes-dev-token`, 보관 7일 |
| MLflow | `infra/mlflow` | `localhost:5000` | 실험 기록과 모델 버전. 예측은 analytics-backend가 수행 |

센서 에뮬레이터 설정 SQLite는 이 저장소에 없습니다.

## 실행

```bash
cp .env.example .env
docker compose up -d --build
docker compose down
```

`compose.yaml`은 더 이상 쓰지 않습니다. `docker compose`는 `docker-compose.yml`만 읽습니다.

MariaDB는 데이터 디렉터리가 비어 있을 때만 `mariadb/init.sql`을 실행합니다. 이미 볼륨이 있으면 `docker compose down -v` 후 다시 띄워야 스키마가 적용됩니다. 앱 백엔드는 기동 시 `create_all`도 하므로, 빈 DB에서는 둘 중 어느 쪽이 먼저여도 같은 테이블이 됩니다.

## 스키마

`mariadb/init.sql`은 운영 중인 MES DB의 `SHOW CREATE TABLE`과 같습니다.

- `operators`, `machines`, `lots`, `work_orders`
- `inspections`, `shipments`, `trace_events`, `alerts`
- `sensor_readings` (스칼라 센서 원시값. 파형 블록은 InfluxDB `waveform` measurement)

## MQTT

토픽 접두사는 `mes`입니다. 브로커 설정은 `mosquitto/mosquitto.conf`입니다.

| 토픽 | 발행 |
| --- | --- |
| `mes/machines/{code}/job` | MES, retained |
| `mes/machines/{code}/state` | edge-gateway |
| `mes/machines/{code}/events` | edge-gateway |
| `mes/machines/{code}/sensors` | machine-emulator |
| `mes/machines/{code}/waveform` | sensor-emulator |
| `mes/machines/{code}/features` | sensor-collector |
| `mes/machines/{code}/anomaly` | analytics-backend |
| `mes/sim/{code}/labels` | sensor-emulator |
| `mes/gateways/{id}/status` | edge-gateway, retained |

개발 설정은 `allow_anonymous true`입니다. 운영에서는 비밀번호 파일과 TLS로 바꿉니다.
