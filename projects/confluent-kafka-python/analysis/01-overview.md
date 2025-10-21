# confluent-kafka-python: 개요

## 1. 기본 정보
- **원본**: [https://github.com/confluentinc/confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python)
- **목적**: Apache Kafka를 위한 고성능, 고기능의 Python 클라이언트를 제공합니다.
- **기술 스택**: Python, C (librdkafka)
- **주요 클래스**: `Producer`, `Consumer`, `AdminClient`, `SchemaRegistryClient`, `AIOProducer`

## 2. 해결하는 문제

`confluent-kafka-python`은 기존의 순수 Python으로 구현된 Kafka 클라이언트가 가진 **성능 문제**를 해결하는 데 중점을 둡니다.

- **문제점**: 순수 Python 클라이언트는 높은 처리량이 요구되는 프로덕션 환경에서 GIL(Global Interpreter Lock)과 네트워크 I/O 처리 방식 때문에 성능 병목이 발생하기 쉽습니다.
- **해결책**: C언어로 작성된 고성능 Kafka 클라이언트 라이브러리인 `librdkafka`를 Python 래퍼(wrapper)로 감싸서 사용합니다. 이를 통해 Python의 사용 편의성은 유지하면서 C 레벨의 성능(높은 처리량, 낮은 지연 시간)을 확보합니다.

결과적으로, 대규모 실시간 데이터 파이프라인을 구축해야 하는 Python 기반 애플리케이션이 프로덕션 환경에서도 안정적으로 Kafka를 사용할 수 있게 해줍니다.

## 3. 왜 이 프로젝트를 선택했나?

- **Kafka 클라이언트의 표준**: Python 생태계에서 사실상의 표준(de-facto standard) Kafka 클라이언트로, 수많은 기업의 프로덕션 환경에서 검증되었습니다.
- **핵심 패턴 학습**: 공식 클라이언트를 통해 Kafka의 Producer/Consumer를 가장 올바르게 사용하는 방법(Best Practice)과 핵심 패턴(예: 에러 핸들링, 재시도, 배치 처리)을 학습할 수 있습니다.
- **성능 트레이드오프 이해**: 순수 Python 구현과 C 라이브러리 기반 구현의 성능 차이와 그 이유를 이해하고, "성능 vs 개발 편의성"이라는 트레이드오프를 어떻게 극복하는지 배울 수 있습니다.
- **비동기 처리**: `asyncio` 지원을 통해 최신 Python 비동기 웹 프레임워크(FastAPI 등)와 Kafka를 통합하는 방법을 학습할 수 있습니다.