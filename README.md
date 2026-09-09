# 사출성형 데이터 가이드 코드 재현

KAMP 사출성형 데이터셋으로 잡음제거 오토인코더와 제품별 준지도 분류를 재현한다. 기존 시간순 불량 예측 실험과 독립된 프로젝트다. 두 프로젝트는 별도 GitHub 레포지토리에 게시한다.

현재 실행 상태: 오토인코더와 CN7/RG3의 SVM·RF·GaussianNB 완료. CN7/RG3 DNN은 제품별 독립 프로세스로 실행 중이며 최종 결과에 아직 포함하지 않았다. 회차마다 모델과 다음 회차 입력을 저장한다. DNN 실행 및 최종 결과 정리는 남아 있다.

## 실행

1. Python 환경에 `requirements.txt`를 설치한다. Colab에서는 제공되는 TensorFlow 환경도 사용할 수 있지만 버전 차이로 수치가 달라질 수 있다.
2. KAMP에서 원본 데이터 파일을 받아 한 폴더에 둔다. 데이터와 PDF는 이 저장소에 포함하지 않는다.
3. `molding_reference.ipynb`의 `DATA_DIR`를 그 폴더로 지정하고 위에서부터 실행한다. Colab에서 Drive를 쓰면 먼저 `from google.colab import drive; drive.mount('/content/drive')`를 실행한다.
4. 결과는 `RESULT_DIR`에 저장한다. GPU는 필수가 아니다. RF 1,200개 트리와 DNN 반복 학습은 CPU에서 시간이 걸린다.

필요한 파일: `labeled_data.csv`, `supervised_label_cn7.csv`, `moldset_labeled_cn7.csv`, `moldset_labeled_rg3.csv`, `moldset_unlabeled_cn7.csv`, `moldset_unlabeled_rg3.csv`.

## 재현 범위

| 단계 | 적용 방식 | 가이드 인쇄 쪽수 |
|---|---|---|
| CN7 전처리 | 설비·제품 선택, LH/RH 연결, 제공 가공 데이터와 대조 | 43–49 |
| 오토인코더 | 입력 dropout .3, 15→5→15, Adam .01, MSE, 최대 30 epoch | 50–51 |
| 이상 판정 | 학습 복원 오차 평균 + 5 × 표준편차 | 55–58 |
| 준지도 입력 | 제품별 제공 moldset 파일, 7:3 층화 분리, seed 42 | 69–74 |
| 준지도 분류 | 확신도 상위 10%씩 추가, 미라벨 약 90% 사용 | 74–77 |
| DNN | 32→64→32→16→1, dropout .25/.2, 반복별 최대 100 epoch | 78–82 |

SVM은 실제 학습 코드의 C=.001, gamma=.01, class_weight={0:100,1:1}을 사용한다. RF는 출력에 기재된 1,200 trees, max_depth=10, min_samples_split=5, bootstrap=False를 적용한다. GaussianNB는 기본값을 사용한다. 예시 GridSearch 전체를 재실행한 프로젝트는 아니다.

## 원문과 달라지는 부분

- 현재 API 호환: Adam `lr`→`learning_rate`, Keras `predict_proba`→`predict`, RF `max_features='auto'`→`'sqrt'`.
- 원문에서 빠진 `without_label` 초기화 및 연결 시 목표값 배열 형태를 명시한다.
- 원문에 없는 신경망/RF seed=42를 추가한다. 과거 라이브러리·초기 가중치는 제공되지 않아 비트 단위 재현을 주장하지 않는다.
- 모델 저장 형식은 현재 Keras의 `.keras`를 사용한다. 출력 진행 표시와 추가 AP/확률 AUC는 학습 규칙을 바꾸지 않는다.
- 원문의 SVM 튜닝 출력과 실제 학습 코드가 다르므로 실제 학습 코드를 우선한다.

입력 대조에서 `supervised_label_cn7.csv`는 24개 공정 변수를 갖지만, 원문 전처리 코드는 `Switch_Over_Position`, `Barrel_Temperature_7`을 제거하지 않아 26개가 남는다. 오토인코더에는 원문 코드대로 26개를 사용한다. 공통 열의 값과 행 순서는 제공 가공 파일과 대조한다. 준지도 DNN은 별도 moldset 파일의 24개 변수를 사용한다.

## 평가 해석

오토인코더는 **원문처럼 정상과 불량에 별도로 MinMaxScaler를 학습**한다. 평가 정상도 정규화 범위 추정에 포함된다. 정답 라벨을 알아야 같은 변환을 선택할 수 있으므로 실제 운영 평가로 해석하면 안 된다.

준지도 실험은 배포된 가공 CSV와 무작위 분리를 그대로 쓴다. 전처리 이전 원자료와의 대응 관계나 시간상 독립성을 보장하지 않는다. CN7 평가 불량은 5건, RG3는 8건으로, 가이드 결과 표의 56건·83건과 다르다. 표본을 임의 복제해서 그 수치에 맞추지 않는다.

따라서 기존 시간순 실험과의 점수 차이를 성능 개선율로 제시하지 않는다. 기존 실험은 시간적 일반화와 검사 부담을 검토하고, 이 저장소는 제공 분석 절차의 재현 가능성과 한계를 확인한다.

## 출처

중소벤처기업부, Korea AI Manufacturing Platform(KAMP), 사출성형기 AI 데이터셋, KAIST(울산과학기술원, ㈜이피엠솔루션즈), 2020.12.14., https://www.kamp-ai.kr/

KAMP의 연구·공식 활용 안내에는 출처 표기와 활용 내용·문서를 kamp@kaist.ac.kr로 보내는 절차가 있다. 이 프로젝트에서 이메일을 발송하지 않았다.
