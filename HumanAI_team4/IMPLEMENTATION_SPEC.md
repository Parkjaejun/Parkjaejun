# 경구알약 불량 탐지 시스템 — 파이프라인 구현 명세서

> **HumanAI Team 4** · 작성일 2026-06-17
> 범위: **서빙 파이프라인** `📱 → 🔥 → 🖥️ → 🔥 → 📱`
> 학습 모델 구현은 별도 문서 → [`MODEL_TRAINING_SPEC.md`](./MODEL_TRAINING_SPEC.md)

---

## 0. 확정 기술 스택 (Decisions)

| 항목 | 결정 | 비고 |
|---|---|---|
| 모바일 앱 | **Flutter** | iOS/Android 단일 코드, Firebase 연동 |
| 추론 서버 | **개발=온프레미스 GTX 3060 / 운영=GCP GPU** | 동일 코드, 환경만 분리 |
| 서버↔Firebase | **Firestore 실시간 구독(pull)** | NAT 통과, 포트개방 불필요 |
| 식별 모델 | YOLO 식별기 (산출물 소비) | 학습은 별도 문서 |
| 이상탐지 | PatchCore / Anomalib (산출물 소비) | 학습은 별도 문서 |

> 본 문서는 **학습된 모델 산출물을 가져다 서빙하는 파이프라인**만 다룬다. 모델을 만드는 방법(데이터·증강·학습·평가)은 `MODEL_TRAINING_SPEC.md` 참조.

---

## 1. 파이프라인 개요

### 1.1 목표
휴대폰으로 촬영한 경구알약 1정의 이미지를 받아 **(1) 약종 식별** → **(2) 정상/불량 판정**하고 결함 위치와 함께 휴대폰으로 반환한다.

### 1.2 전체 흐름 (9단계)
```
📱 휴대폰            🔥 Firebase        🖥️ 서버 · Docker              🔥 Firebase     📱 휴대폰
①촬영 ②업로드+요청  →  (보관·게시)  →  ③감지 ④전처리 ⑤식별 ⑥판정 ⑦업로드  →  ⑧전달  →  ⑨표시
```
| # | 단계 | 담당 |
|---|---|---|
| ① | 카메라 촬영(가이드로 정렬·조명) | 휴대폰 |
| ② | 이미지 업로드 + 요청 문서 생성(`pending`) | 휴대폰 |
| ③ | 새 요청 감지(Firestore 실시간 구독) | 서버 |
| ④ | 이미지 다운로드 + 전처리(검출·crop·정렬) | 서버 |
| ⑤ | YOLO 식별 → 약종 결정(저신뢰=검토) | 서버 |
| ⑥ | 해당 약종 PatchCore 로드 + 불량 판정 | 서버 |
| ⑦ | 결과 이미지 업로드 + 문서 갱신(`done`) | 서버 |
| ⑧ | 결과 전달(실시간/FCM) | Firebase |
| ⑨ | 결과 표시(정상/불량 + 결함 위치) | 휴대폰 |

> 예상 지연: 업로드~결과 **2~5초**(첫 약종은 모델 로드로 +α).

---

## 2. 시스템 아키텍처

### 2.1 3계층 책임
| 계층 | 구성 | 책임 | GPU |
|---|---|---|---|
| 휴대폰 | Flutter 앱 | 촬영·업로드·결과표시 | ✗ |
| Firebase | Storage / Firestore / Auth / FCM | 이미지 보관·요청/결과 중계 | ✗ |
| 서버 | Docker(worker + 모델) | 전처리·추론·모델 라우팅 | ✅ |

### 2.2 개발/운영 환경 분리
| | 개발(온프레미스) | 운영(클라우드) |
|---|---|---|
| 장비 | GTX 3060 PC | GCP Compute Engine GPU(L4/T4) |
| 작업 수신 | Firestore 구독(pull) | Firestore 구독(pull) — **동일** |
| 배포 | `docker compose up` | 동일 이미지 + (선택) GKE/Cloud Run+GPU |
| 비밀키 | 로컬 `secrets/` | Secret Manager |

> worker는 outbound Firestore 구독만 쓰므로 **온프렘이든 클라우드든 동일 코드**. 환경 차이는 `.env`로 흡수. 운영 트래픽 증가 시 Pub/Sub push로 전환 가능(§7.6).

---

## 3. 데이터 계약 (Firebase)

### 3.1 Storage 경로
```
uploads/{userId}/{requestId}.jpg        # 휴대폰 업로드 원본
results/{userId}/{requestId}.png        # 서버 생성 결과(히트맵 오버레이)
```

### 3.2 Firestore 문서 — `requests/{requestId}`
```jsonc
{
  "userId":        "string",
  "imagePath":     "uploads/uid/req.jpg",
  "status":        "pending",           // 상태머신 §3.3
  "createdAt":     "timestamp",
  "updatedAt":     "timestamp",

  // ── 서버가 채우는 결과 필드 ──
  "drugId":        "string|null",
  "drugName":      "string|null",
  "identifyConf":  0.0,
  "verdict":       "normal|defect|review|error|null",
  "anomalyScore":  0.0,
  "threshold":     0.0,
  "defectBoxes":   [],                  // [{x,y,w,h,score}]
  "resultImage":   "results/uid/req.png|null",
  "latencyMs":     0,
  "errorMsg":      "string|null",
  "modelVersion":  "yolo:v1.2,patchcore:v1.0"
}
```

### 3.3 상태 머신 (`status`)
```
pending ──(서버 수신)──▶ processing ──┬──▶ done    (판정 완료)
                                       ├──▶ review  (식별 저신뢰 → 사람 검토)
                                       └──▶ error   (전처리/추론 실패)
```
- 휴대폰: `pending` 생성 후 문서 **실시간 구독**, 종결 상태(`done/review/error`)에서 UI 갱신
- 타임아웃: 앱에서 30초 내 미종결 시 오류 표시

---

## 4. 모바일 앱 명세 (Flutter)

### 4.1 화면 구성
1. **로그인** — Firebase Auth
2. **촬영** — 카메라 프리뷰 + **촬영 가이드 오버레이**(중앙 가이드, 거리·밝기 안내)
3. **확인/업로드** — 미리보기 → 업로드 → 진행 인디케이터
4. **결과** — 정상/불량 배지, 결함 히트맵, 약종명·신뢰도

### 4.2 촬영 가이드 (도메인 갭 완화 — 중요)
- 중앙 가이드 안에 알약 정렬 유도
- 밝기 부족/과다·초점 흐림 시 **셔터 비활성 + 안내**
- 단색 배경 권장 안내

### 4.3 업로드 + 요청 생성 (의사코드)
```dart
final requestId = uuid.v4();
final ref = storage.ref('uploads/$uid/$requestId.jpg');
await ref.putFile(jpgFile, SettableMetadata(contentType: 'image/jpeg'));

await firestore.collection('requests').doc(requestId).set({
  'userId': uid,
  'imagePath': 'uploads/$uid/$requestId.jpg',
  'status': 'pending',
  'createdAt': FieldValue.serverTimestamp(),
});

firestore.collection('requests').doc(requestId).snapshots().listen((doc) {
  final s = doc['status'];
  if (s == 'done' || s == 'review' || s == 'error') showResult(doc);
});
```

### 4.4 주요 패키지
`firebase_core`, `firebase_auth`, `cloud_firestore`, `firebase_storage`, `firebase_messaging`(선택), `camera`, `image_picker`, `image`

### 4.5 클라이언트 경량 전처리
- 업로드 전 **리사이즈(장변 ~1280px) + JPEG 압축(품질 85)**
- EXIF 회전 보정

---

## 5. Firebase 구성

### 5.1 Storage 보안 규칙
```
match /uploads/{uid}/{file} {
  allow write: if request.auth.uid == uid
            && request.resource.size < 8*1024*1024
            && request.resource.contentType.matches('image/.*');
  allow read:  if request.auth.uid == uid;
}
match /results/{uid}/{file} {
  allow read:  if request.auth.uid == uid;
  allow write: if false;          // 서버(Admin SDK)만
}
```

### 5.2 Firestore 보안 규칙
```
match /requests/{id} {
  allow create: if request.auth.uid == request.resource.data.userId
             && request.resource.data.status == 'pending';
  allow read:   if request.auth.uid == resource.data.userId;
  allow update, delete: if false;  // 서버(Admin SDK)만
}
```

### 5.3 서비스 계정
- 서버용 **서비스 계정 키(JSON)** → Admin SDK 인증. 개발 `secrets/sa.json`, 운영 Secret Manager.

---

## 6. 서버 — 디렉토리 & 모듈 구조

```
server/
├── worker/
│   ├── main.py              # Firestore 리스너 진입점
│   ├── pipeline.py          # 전처리→식별→판정 오케스트레이션
│   ├── preprocess.py        # 검출·crop·회전정렬·앞뒷면
│   ├── identifier.py        # YOLO 식별 래퍼 (모델 산출물 로드)
│   ├── anomaly.py           # PatchCore(Anomalib) 래퍼 + 모델 캐시
│   ├── firebase_io.py       # Storage 다운/업로드, Firestore 갱신
│   ├── postprocess.py       # 히트맵 오버레이, 결과 이미지 생성
│   └── config.py            # 환경변수 로딩
├── models/                  # ← 학습 산출물(외부 입력, MODEL_TRAINING_SPEC 참조)
│   ├── yolo/identifier.pt
│   └── patchcore/{drugId}.ckpt
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

> `models/`는 **학습 파이프라인의 산출물**이다. 본 문서는 이를 **로드해 서빙**하는 부분만 규정한다. 생성 방법은 `MODEL_TRAINING_SPEC.md`.

### 6.1 주요 의존성
```
torch>=2.2  torchvision
anomalib>=1.1
ultralytics            # 또는 mmyolo
firebase-admin
opencv-python  numpy  pillow
redis                  # GPU 직렬화 큐(선택)
```

---

## 7. 서버 — 추론 파이프라인 명세

### 7.1 리스너 (`main.py`)
```python
# pending 문서 실시간 구독 (outbound → NAT 통과)
query = db.collection('requests').where('status', '==', 'pending')
query.on_snapshot(on_new_requests)   # 신규 → enqueue
```
- 단일 3060 → **Redis 큐로 직렬 처리**(동시 추론 1건)
- 멱등성: 처리 시작 시 트랜잭션으로 `status=processing` 선점 → 중복 방지

### 7.2 파이프라인 (`pipeline.py`)
```python
def handle(req):
    img = fb.download(req['imagePath'])              # ③④
    roi = preprocess(img)
    if roi is None:
        return finish(req, status='error', errorMsg='no_pill_detected')

    drug_id, name, conf = identifier.identify(roi)   # ⑤
    if conf < CONF_THRESH:
        return finish(req, status='review', identifyConf=conf)

    model = anomaly_cache.get(drug_id)               # ⑥ 모델 라우팅
    score, heatmap, boxes = model.predict(roi)
    verdict = 'defect' if score >= model.threshold else 'normal'

    url = fb.upload_result(req, overlay(roi, heatmap, boxes))  # ⑦
    finish(req, status='done', drugId=drug_id, drugName=name,
           identifyConf=conf, verdict=verdict, anomalyScore=score,
           threshold=model.threshold, defectBoxes=boxes, resultImage=url)
```

### 7.3 전처리 (`preprocess.py`)
1. **알약 검출** — YOLO/분할로 알약 bbox 1개
2. **crop** — bbox + 여백
3. **회전 정렬** — 마스크 PCA 주축 수평화
4. **앞/뒷면** — (있으면) 분류 → 모델 선택 반영
5. **정규화** — 화이트밸런스/조도 보정, 리사이즈(예: 256/320px)
> 정렬·보정 없으면 PatchCore 오판율 급증 → **필수**.

### 7.4 모델 캐시 / 라우팅 (`anomaly.py`)
```python
class AnomalyCache:
    def __init__(self, dir, max_keep=20):
        self.dir, self.lru = dir, OrderedDict()
    def get(self, drug_id):
        if drug_id in self.lru:
            self.lru.move_to_end(drug_id); return self.lru[drug_id]
        model = Patchcore.load(f'{self.dir}/{drug_id}.ckpt')   # 지연 로딩
        self.lru[drug_id] = model
        if len(self.lru) > self.max_keep: self.lru.popitem(last=False)
        return model
```
- **콜드스타트**: 첫 약종 로드 +0.5~2초. 자주 쓰는 약종 **pre-warm**.
- 임계값(threshold)은 ckpt 메타에 포함(학습 단계 산출).

### 7.5 후처리 (`postprocess.py`)
- heatmap을 ROI에 오버레이 → 결함 위치 PNG 생성
- 임계 초과 영역을 `defectBoxes`로 추출

### 7.6 운영 확장 (선택)
- 트래픽 증가 시: Storage 업로드 → Cloud Function → **Pub/Sub** → GPU 워커 풀(GKE). pull 리스너 유지하며 점진 전환.

---

## 8. Docker / 배포

### 8.1 Dockerfile (요지)
```dockerfile
FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04
RUN apt-get update && apt-get install -y python3 python3-pip libgl1
COPY requirements.txt .
RUN pip3 install -r requirements.txt
COPY worker/ /app/worker
WORKDIR /app
CMD ["python3", "-m", "worker.main"]
```

### 8.2 docker-compose.yml (개발=온프렘)
```yaml
services:
  worker:
    build: .
    env_file: .env
    volumes:
      - ./models:/app/models
      - ./secrets:/app/secrets:ro
    deploy:
      resources:
        reservations:
          devices: [{driver: nvidia, count: 1, capabilities: [gpu]}]
    depends_on: [redis]
  redis:
    image: redis:7
```
- 사전 요구: 호스트에 **NVIDIA Driver + NVIDIA Container Toolkit**

### 8.3 운영(GCP)
- GPU VM(L4/T4)에 동일 이미지 배포(Artifact Registry → `docker run --gpus all`)
- 비밀키 Secret Manager, 모델 GCS→로컬 동기화 또는 이미지 포함
- worker 코드 변경 없음(Firestore 구독 동일)

### 8.4 환경변수(`.env`)
```
GOOGLE_APPLICATION_CREDENTIALS=/app/secrets/sa.json
FIREBASE_PROJECT_ID=...
STORAGE_BUCKET=...
CONF_THRESH=0.6
MODEL_CACHE_SIZE=20
REDIS_URL=redis://redis:6379
```

---

## 9. 보안
- Auth 필수, Storage/Firestore 규칙으로 본인 데이터만 접근
- 서비스 계정 키는 서버에만(앱에 포함 금지)
- 요청·결과 이미지 PII 미포함 확인

---

## 10. E2E 테스트 (파이프라인)
- 휴대폰→Firebase→서버→결과까지 **통합 시나리오** 자동화
- 검증 항목: 정상 경로(`done`), 검토 경로(`review`), 오류 경로(`error`), 콜드스타트/캐시히트 지연(p50/p95), 타임아웃
- 부하: 동시 요청 큐잉·직렬 처리 정상성
> 모델 정확도 지표(AUROC·Recall 등)는 `MODEL_TRAINING_SPEC.md`에서 다룸.

---

## 11. 미결정 사항 (Open Questions)
1. 앞/뒷면 처리: 면별 모델 vs 양면 통합 — 데이터 확인 후
2. 인증 방식: 익명 vs 계정 기반
3. 결과/업로드 이미지 보존·삭제 정책
4. 운영 동시 사용자 규모 → GPU 대수/스케일 전략

---

## 12. 리스크 (파이프라인 관점)
| 리스크 | 대응 |
|---|---|
| **도메인 갭**(스튜디오↔휴대폰) | 촬영 가이드, 전처리 정렬·조도 보정 |
| 정렬 실패 → 오판 | 전처리 정렬 필수, 실패 시 `review` |
| 콜드스타트 지연 | pre-warm, LRU 캐시 |
| NAT/네트워크 | Firestore 구독(outbound), 재연결 로직 |
| 200 모델 관리 | 모델 버전관리, 클러스터 묶음 옵션(학습 문서) |

---

*본 문서는 서빙 파이프라인만 규정한다. 모델 학습·평가·증강·합성불량은 [`MODEL_TRAINING_SPEC.md`](./MODEL_TRAINING_SPEC.md) 참조.*
