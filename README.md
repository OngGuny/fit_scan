# 👕 FitScan — AI 기반 영상 의류 분석 플랫폼

> 영상에서 의류를 자동 감지하고, 색상·사이즈·카테고리를 추론하는 Computer Vision + AI Pipeline

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![YOLOv8](https://img.shields.io/badge/YOLOv8-Ultralytics-FF6F00)](https://docs.ultralytics.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## 📌 프로젝트 개요

FitScan은 업로드된 영상(MP4 등)에서 프레임을 추출하고, 딥러닝 모델을 활용해 **의류를 자동 감지·분류**한 뒤 **색상 분석**, **사이즈 추정**, **카테고리 분류** 결과를 제공하는 PoC 수준의 AI 영상 분석 플랫폼입니다.

### 핵심 기능

- **의류 감지 (Detection)** — YOLOv8 기반 Object Detection으로 영상 내 의류 아이템 위치 및 바운딩 박스 검출
- **색상 분석 (Color Analysis)** — 세그멘테이션 영역 기반 K-Means 클러스터링으로 의류 주요 색상(Dominant Color) 추출
- **사이즈 추정 (Size Estimation)** — Pose Estimation + 의류 키포인트를 활용한 상대적 사이즈(S/M/L/XL) 추론
- **카테고리 분류 (Classification)** — 상의/하의/아우터/원피스 등 의류 카테고리 자동 분류
- **분석 리포트** — 분석 결과를 구조화된 JSON 및 대시보드 UI로 제공

---

## 🏗️ 시스템 아키텍처

```mermaid
flowchart TB
    subgraph Client["🖥️ Client Layer"]
        UI["Dashboard UI\n(React + Vite)"]
        API_CLIENT["REST API Consumer"]
    end

    subgraph API["⚡ API Layer — FastAPI"]
        UPLOAD["/api/v1/videos/upload"]
        ANALYZE["/api/v1/videos/{id}/analyze"]
        RESULT["/api/v1/reports/{id}"]
        WS["WebSocket\n/ws/analysis/{id}"]
    end

    subgraph Pipeline["🔬 Analysis Pipeline"]
        direction TB
        EXTRACT["Frame Extractor\n(OpenCV)"]
        DETECT["Clothing Detector\n(YOLOv8)"]
        SEGMENT["Clothing Segmenter\n(Mask R-CNN / SAM)"]
        COLOR["Color Analyzer\n(K-Means + HSV)"]
        SIZE["Size Estimator\n(MediaPipe Pose)"]
        CLASSIFY["Category Classifier\n(ResNet / EfficientNet)"]
        AGGREGATE["Result Aggregator"]
    end

    subgraph Storage["💾 Storage Layer"]
        PG[("PostgreSQL\n분석 결과/메타데이터")]
        MINIO["MinIO / Local FS\n영상·프레임 저장"]
        REDIS[("Redis\n작업 큐/캐시")]
    end

    subgraph Worker["⚙️ Task Worker"]
        CELERY["Celery Worker"]
    end

    UI --> UPLOAD & RESULT & WS
    API_CLIENT --> ANALYZE & RESULT
    UPLOAD --> REDIS
    ANALYZE --> REDIS
    REDIS --> CELERY
    CELERY --> EXTRACT --> DETECT --> SEGMENT
    SEGMENT --> COLOR & SIZE
    COLOR --> AGGREGATE
    SIZE --> AGGREGATE
    DETECT --> CLASSIFY --> AGGREGATE
    AGGREGATE --> PG
    EXTRACT --> MINIO
    RESULT --> PG
    WS -.->|진행률 Push| UI
```

---

## 🔄 분석 파이프라인 상세

```mermaid
flowchart LR
    A["📹 영상 업로드\n(MP4, AVI, MOV)"] --> B["🎞️ 프레임 추출\nOpenCV\nN fps 샘플링"]
    B --> C["👔 의류 감지\nYOLOv8\nDeepFashion2 FT"]
    C --> D["✂️ 세그멘테이션\nMask R-CNN\nor SAM"]
    D --> E["🎨 색상 분석"]
    D --> F["📏 사이즈 추정"]
    C --> G["🏷️ 카테고리 분류"]
    E & F & G --> H["📊 결과 집계\n& 리포트 생성"]
    H --> I["💾 DB 저장\n& API 응답"]
```

### Stage별 기술 상세

#### 1. 프레임 추출 (Frame Extraction)

| 항목 | 내용 |
|------|------|
| 라이브러리 | OpenCV (`cv2.VideoCapture`) |
| 샘플링 전략 | 설정 가능한 FPS 기반 균등 샘플링 (기본: 1fps) |
| 키프레임 감지 | Scene Change Detection으로 중복 프레임 제거 |
| 출력 | JPEG 프레임 이미지 + 타임스탬프 메타데이터 |

#### 2. 의류 감지 (Clothing Detection)

| 항목 | 내용 |
|------|------|
| 모델 | YOLOv8n/s (Ultralytics) |
| 학습 데이터 | DeepFashion2 / ModaNet Fine-tuning |
| 감지 대상 | 상의, 하의, 아우터, 원피스, 스커트 등 13개 카테고리 |
| 출력 | 바운딩 박스 좌표, confidence score, 클래스 라벨 |

#### 3. 의류 세그멘테이션 (Segmentation)

| 항목 | 내용 |
|------|------|
| 모델 옵션 A | Mask R-CNN (Detectron2 기반) |
| 모델 옵션 B | SAM (Segment Anything Model) — 바운딩 박스 프롬프트 방식 |
| 목적 | 의류 영역만 정밀하게 마스킹하여 배경 노이즈 제거 |
| 출력 | 픽셀 단위 세그멘테이션 마스크 |

#### 4. 색상 분석 (Color Analysis)

```mermaid
flowchart LR
    MASK["세그멘테이션\n마스크 적용"] --> HSV["BGR → HSV\n색공간 변환"]
    HSV --> KMEANS["K-Means\nClustering\n(k=3~5)"]
    KMEANS --> DOMINANT["Dominant Color\n추출 (Top-3)"]
    DOMINANT --> NAMING["색상 네이밍\n(CSS Named Colors\n매핑)"]
    NAMING --> RATIO["색상 비율\n계산"]
```

| 항목 | 내용 |
|------|------|
| 전처리 | 세그멘테이션 마스크로 의류 영역만 추출 + 가우시안 블러 노이즈 제거 |
| 색공간 | HSV 변환 → Hue 기반 색상 그룹핑에 유리 |
| 클러스터링 | scikit-learn K-Means (k=3~5, 의류별 adaptive) |
| 색상 네이밍 | HSV 범위 → 한국어/영어 색상명 매핑 테이블 |
| 출력 | `[{color: "#2E4057", name: "네이비", ratio: 0.65}, ...]` |

#### 5. 사이즈 추정 (Size Estimation)

```mermaid
flowchart LR
    POSE["MediaPipe Pose\n33 Keypoints"] --> LANDMARK["어깨·허리·엉덩이\n키포인트 추출"]
    BBOX["의류 바운딩 박스\n(YOLO)"] --> RELATIVE["신체 대비\n의류 비율 계산"]
    LANDMARK --> RELATIVE
    RELATIVE --> SIZE_MAP["비율 → 사이즈\n매핑 테이블\n(S/M/L/XL)"]
    SIZE_MAP --> CONFIDENCE["추정 신뢰도\n산출"]
```

| 항목 | 내용 |
|------|------|
| 포즈 추정 | MediaPipe Pose (33 landmarks, 실시간 추론 가능) |
| 핵심 키포인트 | 어깨 너비, 상체 길이, 허리-엉덩이 비율 |
| 사이즈 추론 방식 | 신체 키포인트 기반 상대 비율 → 사이즈 등급 매핑 |
| 한계 | 2D 기반 추론이므로 정밀도 한계 존재 → 신뢰도(confidence) 함께 제공 |
| 출력 | `{estimated_size: "L", confidence: 0.78, shoulder_ratio: 1.23}` |

#### 6. 카테고리 분류 (Classification)

| 항목 | 내용 |
|------|------|
| 모델 | EfficientNet-B0 또는 ResNet-50 (Transfer Learning) |
| 학습 데이터 | DeepFashion2 카테고리 라벨 |
| 분류 체계 | 상의(티셔츠/셔츠/니트), 하의(데님/슬랙스/숏), 아우터(자켓/코트), 원피스, 스커트 |
| 출력 | `{category: "상의", subcategory: "티셔츠", confidence: 0.92}` |

---

## 🛠️ 기술 스택

### Backend

| 기술 | 용도 | 선정 이유 |
|------|------|-----------|
| **Python 3.11+** | 메인 런타임 | CV/ML 생태계 최적, 타입 힌트 활용 |
| **FastAPI** | API 서버 | 비동기 지원, 자동 OpenAPI 문서, Pydantic 검증 |
| **Celery + Redis** | 비동기 작업 큐 | 영상 분석은 Heavy Task → 비동기 처리 필수 |
| **PostgreSQL** | 분석 결과 저장 | JSONB로 유연한 분석 결과 저장, pgvector 확장 가능 |
| **MinIO / Local FS** | 파일 스토리지 | 영상·프레임 이미지 저장, S3 호환 |
| **SQLAlchemy 2.0** | ORM | 비동기 세션, 타입 안전성 |
| **Alembic** | DB 마이그레이션 | 스키마 버전 관리 |

### AI / ML

| 기술 | 용도 | 선정 이유 |
|------|------|-----------|
| **YOLOv8 (Ultralytics)** | 의류 감지 | SOTA 실시간 Object Detection, 파인튜닝 용이 |
| **Mask R-CNN / SAM** | 세그멘테이션 | 픽셀 단위 의류 영역 추출 |
| **MediaPipe Pose** | 포즈 추정 | 경량·실시간, 33 keypoints |
| **EfficientNet / ResNet** | 분류 | Transfer Learning 효율, 검증된 아키텍처 |
| **OpenCV** | 영상 처리 | 프레임 추출, 이미지 전처리, 색공간 변환 |
| **scikit-learn** | 클러스터링 | K-Means 색상 클러스터링 |
| **PyTorch** | 모델 학습/추론 | CUDA 지원, 커뮤니티 활발 |

### Frontend (대시보드)

| 기술 | 용도 |
|------|------|
| **React + Vite** | SPA 대시보드 |
| **TailwindCSS** | 스타일링 |
| **Recharts / Chart.js** | 색상 분포 차트, 분석 통계 시각화 |
| **React Query** | 서버 상태 관리 |

### Infra / DevOps

| 기술 | 용도 |
|------|------|
| **Docker Compose** | 로컬 개발 환경 구성 (API + Worker + DB + Redis + MinIO) |
| **GitHub Actions** | CI 파이프라인 (린트, 테스트, 이미지 빌드) |

---

## 📂 프로젝트 구조

```
fitscan/
├── README.md
├── docker-compose.yml
├── pyproject.toml
│
├── backend/
│   ├── app/
│   │   ├── main.py                  # FastAPI 엔트리포인트
│   │   ├── config.py                # 환경 설정 (Pydantic Settings)
│   │   │
│   │   ├── api/
│   │   │   └── v1/
│   │   │       ├── videos.py        # 영상 업로드/분석 엔드포인트
│   │   │       └── reports.py       # 분석 리포트 조회 엔드포인트
│   │   │
│   │   ├── domain/
│   │   │   ├── models.py            # SQLAlchemy 모델
│   │   │   └── schemas.py           # Pydantic 스키마
│   │   │
│   │   ├── pipeline/
│   │   │   ├── orchestrator.py      # 파이프라인 오케스트레이터
│   │   │   ├── frame_extractor.py   # 프레임 추출
│   │   │   ├── clothing_detector.py # YOLOv8 의류 감지
│   │   │   ├── segmenter.py         # 세그멘테이션
│   │   │   ├── color_analyzer.py    # 색상 분석
│   │   │   ├── size_estimator.py    # 사이즈 추정
│   │   │   └── classifier.py        # 카테고리 분류
│   │   │
│   │   ├── workers/
│   │   │   ├── celery_app.py        # Celery 설정
│   │   │   └── tasks.py             # 비동기 분석 태스크
│   │   │
│   │   └── infra/
│   │       ├── database.py          # DB 세션 관리
│   │       └── storage.py           # 파일 스토리지 클라이언트
│   │
│   ├── models/                      # 학습된 모델 가중치
│   ├── alembic/                     # DB 마이그레이션
│   └── tests/
│
└── frontend/
    ├── src/
    │   ├── pages/
    │   │   ├── Upload.tsx           # 영상 업로드 페이지
    │   │   ├── Analysis.tsx         # 분석 진행/결과 페이지
    │   │   └── Reports.tsx          # 리포트 목록/상세
    │   └── components/
    │       ├── ColorPalette.tsx      # 색상 분포 시각화
    │       ├── SizeEstimate.tsx      # 사이즈 추정 표시
    │       └── DetectionOverlay.tsx  # 바운딩 박스 오버레이
    └── package.json
```

---

## 🔌 API 설계

### 핵심 엔드포인트

```
POST   /api/v1/videos/upload          영상 파일 업로드
POST   /api/v1/videos/{id}/analyze    분석 시작 (비동기)
GET    /api/v1/videos/{id}/status     분석 진행률 조회
GET    /api/v1/reports/{id}           분석 리포트 조회
GET    /api/v1/reports                리포트 목록 (페이지네이션)
WS     /ws/analysis/{id}             분석 진행률 실시간 Push
```

### 분석 리포트 응답 예시

```json
{
  "report_id": "rpt_abc123",
  "video_id": "vid_xyz789",
  "analyzed_at": "2026-03-16T14:30:00Z",
  "total_frames_analyzed": 120,
  "items": [
    {
      "item_id": "item_001",
      "category": {
        "main": "상의",
        "sub": "티셔츠",
        "confidence": 0.94
      },
      "colors": [
        {"hex": "#2E4057", "name": "네이비", "ratio": 0.65},
        {"hex": "#FFFFFF", "name": "화이트", "ratio": 0.25},
        {"hex": "#C0392B", "name": "레드", "ratio": 0.10}
      ],
      "estimated_size": {
        "label": "L",
        "confidence": 0.78,
        "metrics": {
          "shoulder_width_ratio": 1.23,
          "torso_length_ratio": 0.95
        }
      },
      "detection": {
        "bbox": [120, 80, 340, 420],
        "confidence": 0.91,
        "frame_timestamp": "00:00:03.500"
      },
      "thumbnail_url": "/api/v1/frames/frm_001/thumbnail"
    }
  ],
  "summary": {
    "total_items_detected": 3,
    "dominant_color": "#2E4057",
    "size_distribution": {"M": 1, "L": 2}
  }
}
```

---

## 📊 데이터 모델

```mermaid
erDiagram
    VIDEO {
        uuid id PK
        string filename
        string storage_path
        int duration_seconds
        int total_frames
        string status "pending|processing|done|failed"
        timestamp created_at
    }

    ANALYSIS_REPORT {
        uuid id PK
        uuid video_id FK
        int frames_analyzed
        jsonb summary
        timestamp analyzed_at
    }

    DETECTED_ITEM {
        uuid id PK
        uuid report_id FK
        string category_main
        string category_sub
        float category_confidence
        jsonb colors "배열: hex, name, ratio"
        string estimated_size
        float size_confidence
        jsonb size_metrics
        jsonb bbox "x1, y1, x2, y2"
        float detection_confidence
        string frame_timestamp
        string thumbnail_path
    }

    VIDEO ||--o{ ANALYSIS_REPORT : "1:N"
    ANALYSIS_REPORT ||--o{ DETECTED_ITEM : "1:N"
```

---

## 🚀 실행 방법

### Prerequisites

- Docker & Docker Compose
- NVIDIA GPU + CUDA (권장, CPU 추론도 가능)
- Python 3.11+

### Quick Start

```bash
# 1. 레포 클론
git clone https://github.com/your-username/fitscan.git
cd fitscan

# 2. 환경 변수 설정
cp .env.example .env

# 3. Docker Compose로 전체 스택 실행
docker compose up -d

# 4. 모델 가중치 다운로드
python scripts/download_models.py

# 5. DB 마이그레이션
alembic upgrade head

# 6. 브라우저에서 대시보드 접속
# http://localhost:3000
```

---

## 📈 개발 로드맵

```mermaid
gantt
    title FitScan 개발 로드맵
    dateFormat YYYY-MM-DD
    axisFormat %m/%d

    section Phase 1 — Foundation
    프로젝트 셋업 & Docker 환경        :done, p1_1, 2026-03-16, 3d
    FastAPI 보일러플레이트 & DB 설계    :p1_2, after p1_1, 3d
    영상 업로드 & 프레임 추출           :p1_3, after p1_2, 3d

    section Phase 2 — Core Pipeline
    YOLOv8 의류 감지 통합              :p2_1, after p1_3, 5d
    세그멘테이션 모듈 (SAM)            :p2_2, after p2_1, 4d
    색상 분석 모듈                     :p2_3, after p2_1, 3d
    사이즈 추정 모듈                   :p2_4, after p2_2, 4d

    section Phase 3 — Integration
    Celery 비동기 파이프라인            :p3_1, after p2_3, 3d
    분석 리포트 API                    :p3_2, after p3_1, 2d
    대시보드 UI 구현                   :p3_3, after p3_2, 5d

    section Phase 4 — Polish
    E2E 테스트 & 정확도 검증           :p4_1, after p3_3, 3d
    문서화 & 데모 영상                 :p4_2, after p4_1, 2d
```

---

## 🔬 기술적 챌린지 & 접근법

### 1. 영상에서의 의류 감지 정확도

**문제**: 움직임, 각도 변화, 부분 가림(occlusion) 등으로 단일 프레임 감지 신뢰도가 떨어질 수 있음

**접근**: 다중 프레임 앙상블 — 시간축으로 감지 결과를 집계하여 최종 결과의 robustness 확보. 동일 아이템의 여러 프레임 감지 결과를 NMS(Non-Maximum Suppression) + IoU 기반으로 병합

### 2. 정확한 색상 추출

**문제**: 조명 조건, 화이트 밸런스, 압축 아티팩트에 따라 동일한 의류도 색상이 달라 보임

**접근**: HSV 색공간 변환 후 클러스터링, 배경 마스킹 통한 노이즈 제거, 다중 프레임 색상 중앙값(median) 사용

### 3. 2D 기반 사이즈 추정의 한계

**문제**: 카메라 거리, 각도, 체형에 따라 2D 비율만으로 절대 사이즈 추론이 부정확

**접근**: 절대 수치가 아닌 **상대적 사이즈 등급**(S/M/L/XL)으로 추론 범위를 제한하고, 신뢰도(confidence)를 함께 제공하여 불확실성을 투명하게 전달

---

## 🎯 이 프로젝트가 보여주는 역량

| 역량 | 내용 |
|------|------|
| **AI Pipeline 설계** | 영상 → 감지 → 세그멘테이션 → 분석의 멀티스테이지 파이프라인 아키텍처 |
| **비동기 처리** | Celery + Redis 기반 Heavy Task 비동기 오케스트레이션 |
| **Computer Vision** | Object Detection, Segmentation, Pose Estimation, Color Analysis 통합 |
| **API 설계** | RESTful API + WebSocket 실시간 피드백 구현 |
| **데이터 모델링** | 영상-리포트-감지아이템 간 관계형 스키마 + JSONB 유연 저장 |
| **프로덕션 지향** | Docker Compose 환경, CI 파이프라인, 모니터링 고려 |

---

## 📚 참고 자료

- [Ultralytics YOLOv8 Documentation](https://docs.ultralytics.com)
- [DeepFashion2 Dataset](https://github.com/switchablenorms/DeepFashion2)
- [ModaNet Dataset](https://github.com/eBay/modanet)
- [MediaPipe Pose](https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker)
- [Segment Anything Model (SAM)](https://segment-anything.com/)
- [MMFashion Toolbox](https://github.com/open-mmlab/mmfashion)

