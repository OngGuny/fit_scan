# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

FitScan은 영상에서 의류를 자동 감지하고 색상·사이즈·카테고리를 추론하는 Event-Driven AI Pipeline 플랫폼이다. 프로덕션 레벨을 목표로 설계·구현 중이며, PoC가 아닌 실서비스 품질 기준을 적용한다.

## Architecture

Event-Driven + DAG Orchestration 패턴:
- **FastAPI** → REST/WebSocket API 서버
- **Kafka** → 분석 스테이지 간 이벤트 디커플링 (partition key: `video_id`)
- **Airflow** → 7-task DAG 오케스트레이션 (KafkaSensor → extract → detect → segment/classify 병렬 → aggregate)
- **PostgreSQL** → 메타데이터/결과 저장 (SQLAlchemy 2.0 + Alembic)
- **Redis** → 캐싱/실시간 진행률
- **MinIO** → S3 호환 파일 스토리지 (영상/프레임)

### ML Pipeline Flow

```
extract_frames (OpenCV)
  → detect_clothing (YOLOv8)
    → [parallel branch]
      ├─ segment (Mask R-CNN/SAM) → analyze_color (K-Means) + estimate_size (MediaPipe Pose)
      └─ classify_category (EfficientNet/ResNet-50)
    → aggregate_results
```

### Kafka Topics

`video.uploaded` → `frames.extracted` → `clothing.detected` → `clothing.segmented` → `analysis.completed` / `analysis.progress`

## Tech Stack

- **Backend**: Python 3.11+, FastAPI 0.110+, SQLAlchemy 2.0, Alembic
- **ML/CV**: YOLOv8 (Ultralytics), Mask R-CNN/SAM, MediaPipe Pose, EfficientNet, OpenCV, scikit-learn, PyTorch
- **Infra**: Kafka 3.7+, Airflow 2.9+, PostgreSQL, Redis, MinIO, Docker Compose
- **Frontend**: React + Vite, TailwindCSS, Recharts/Chart.js, React Query

## Planned Commands

```bash
docker compose up -d                    # 전체 서비스 기동
python scripts/download_models.py       # 모델 가중치 다운로드
alembic upgrade head                    # DB 마이그레이션
```

## Key Design Decisions

- Airflow DAG에서 병렬 브랜치 사용: color/size 분석은 segmentation 이후 동시 실행
- `none_failed_min_one_success` trigger rule로 부분 실패 허용 (color 실패해도 size는 진행)
- XCom은 경량 메타데이터만 전달, 대용량 데이터는 MinIO 경유
- 2D 기반 사이즈 추정의 한계를 confidence score로 보완

## GitHub

- Remote: `https://github.com/OngGuny/fit_scan.git`
- gh auth 계정: `OngGuny` (private repo)
