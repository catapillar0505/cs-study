# ai — 인공지능 기초와 운영

모델 구조(CNN · Attention)와, 그 모델을 실제로 서빙할 때 부딪히는 것들(ONNX · GPU).

| # | 문서 | 다루는 내용 |
|---|---|---|
| 01 | [CNN](01-cnn.md) | 국소 연결과 가중치 공유, 계층적 특징, 전역 맥락의 한계 |
| 02 | [Attention](02-attention.md) | Q·K·V, CNN과의 차이, 계산량과 위치 정보 문제 |
| 03 | [ONNX](03-onnx.md) | 모델 교환 포맷, 내보내기의 함정, 런타임이 하는 일 |
| 04 | [ONNX Runtime — 세션과 스레드](04-onnx-runtime-세션과-스레드.md) | 세션이 필요한 이유, intra/inter op 스레드, OpenMP 오해 |
| 05 | [GPU 인프라](05-gpu-인프라.md) | 왜 나눠 쓰기 어려운가, all-reduce, gang scheduling, MIG |
| 06 | [모델 가중치와 배포](06-모델-가중치-배포.md) | 가중치란 무엇인가, 이미지와 분리해서 배포하는 이유 |
| 07 | [객체 탐지 — 임계값과 NMS](07-객체탐지-임계값과-nms.md) | 신뢰도 임계값의 트레이드오프, IoU와 NMS |
