# CLAUDE_kor.md

이 파일은 이 저장소에서 작업할 때 Claude Code (claude.ai/code)에 제공되는 안내문입니다.
([CLAUDE.md](CLAUDE.md)의 한국어 번역본입니다. 내용을 수정할 때는 두 파일을 함께 갱신하세요.)

## 이 저장소는 무엇인가

아이오와주 USGS 관측소(gauge station)를 대상으로 한 그래프 신경망 기반 하천유량(streamflow/discharge)
예측 및 결측 보간(imputation)을 위한 **데이터 전처리 코드**입니다. **모델도 데이터도 들어 있지 않습니다** —
모델은 [tsl (torch-spatiotemporal)](https://github.com/torchspatiotemporal/tsl)에서 가져오고, 실험용
데이터는 별도로 받아야 합니다(README.md 참고). 이 저장소의 모든 코드는 관측소별 원시 CSV를 tsl이
입력으로 받는 `(target, connectivity, mask)` 3종 세트로 변환하는 일을 합니다.

## 실행 방법

빌드·테스트·린트 설정은 없습니다 — 독립 실행 스크립트와 탐색용 노트북들입니다.

conda 환경 `hydrotgnn`에 의존성이 설치되어 있습니다(tsl 0.9.6, torch, pandas, networkx, pytables,
scikit-learn, matplotlib/seaborn).

스크립트와 노트북의 모든 경로는 스크립트 위치가 아니라 **현재 작업 디렉터리 기준 상대 경로**이며,
이들이 기대하는 데이터 디렉터리는 저장소에 없습니다. 따라서 데이터 루트에서 실행하세요. 예:

```bash
cd <data-root>                 # data_time_series/ 와 catchment_relationship.csv 가 있어야 함
python .../tsl_customed_data_loader/merge_river_data.py   # -> merged_discharge_data.csv
python .../tsl_customed_data_loader/adj_matrix.py         # -> adj_matrix.csv
python .../tsl_customed_data_loader/tsl_data_loader.py    # SpatioTemporalDataset 생성
```

## 데이터 규약 (중요 — 이 저장소의 모든 코드가 공유)

- **관측소별 시계열**: `data_time_series/<station_id>_data.csv`. `datetime` 열, `discharge` 열,
  그리고 `precipitation`부터 `silty_clay_loam`까지 연속된 피처 블록을 가집니다
  (노트북은 이를 위치 기반으로 `X.loc[:, 'precipitation':'silty_clay_loam']`처럼 잘라 씁니다).
- **토폴로지**: `catchment_relationship.csv`. 열은 `station_id`(*하류* 관측소)와 `upstream_id`입니다.
  엣지 방향은 **상류 → 하류**: `adj[upstream, downstream] = 1`.
  스크립트(`adj_matrix.py`)와 노트북 3(`dist.loc[upstream_id, station_id] = 1`) 모두 이 방향을
  사용하므로, 코드를 추가할 때도 이 방향을 유지하세요.
- **관측소 ID는 문자열**이며, 딕셔너리 키와 DataFrame 열 이름으로 쓰입니다. 관계 파일은 반드시
  `dtype=str`로 읽으세요. 그렇지 않으면 `adj_matrix.py`의 `station_to_index` 조회가
  (int 키 vs. str 키 불일치로) 엣지를 조용히 누락시키고 결국 전부 0인 인접행렬이 만들어집니다.
- **결측 유량은 NaN이 아니라 `<= 0`으로 인코딩**되어 있습니다. 노트북들은 이 값을 스캔해 모든
  관측소에서 데이터가 완전한 타임스탬프를 찾고, 살아남은 파일명을 `data.txt`에 기록합니다.
- **열 순서가 중요합니다**: 인접행렬의 행/열 순서는 `merged_discharge_data.csv`의 열 순서를 따릅니다.
  두 파일은 항상 함께 다시 생성하세요.

## 두 가지 파이프라인

### 1. `tsl_customed_data_loader/` — 현재 사용 중, 스크립트 기반 (wide CSV → tsl)

순차적으로 실행되며, 각 단계는 이전 단계의 출력 파일을 입력으로 씁니다:

1. `merge_river_data.py` — 모든 `*_data.csv`를 하나의 wide 프레임(인덱스 `datetime`, 관측소당 한 열)으로
   피벗하며 **discharge만** 담습니다. 날짜의 `/`를 `-`로 정규화하고, 중복 타임스탬프를 제거한 뒤,
   관측소들을 outer join하고, 마지막으로 `interpolate() → ffill → bfill`로 결측을 채웁니다.
   이 보간은 무조건 수행되므로 이후 코드에서는 실측값과 채워넣은 값을 구분할 수 없습니다 —
   유효성 마스크가 필요하다면 채우기 이전에 확보해 두세요.
2. `adj_matrix.py` — 관계 테이블로부터 밀집(dense) 방향성 인접행렬을 만들며, 병합된 CSV의 열 순서에
   맞춰 정렬합니다.
3. `tsl_data_loader.py` — 값을 표준화한 뒤 `SpatioTemporalDataset`으로 감쌉니다
   (`window=24`, `horizon=12`). 거친 부분: 데이터를 이미 손으로 스케일링해 놓고 *또한* 스케일러를
   `transform=`으로 넘기는데, tsl 0.9에서 `transform=`은 샘플 단위 콜러블이지 스케일러 훅
   (`scalers={'target': ...}`)이 아닙니다. 역정규화 결과를 신뢰하기 전에 이 부분을 고치세요.

### 2. `codes/` — 이전 방식, 노트북 기반 (타임스탬프별 CSV → `water.h5`)

탐색용이며, 윈도우 시절의 절대 경로가 하드코딩되어 있고 일부 중국어 주석과 죽은 셀
(예: 노트북 1의 `pymysql` 블록)이 있습니다. 실행 가능한 코드라기보다 HDF5 데이터셋이 어떻게
만들어졌는지에 대한 기록으로 읽으세요.

- `1readindata.ipynb` — 관측소 목록과 관계 파일 점검.
- `2buildgraph.ipynb` — **레이아웃을 전치**합니다: 타임스탬프마다 CSV 한 개
  (`time_series/<datetime>.csv`)를 쓰고, 그 안에 해당 시점의 모든 관측소 행을 담습니다.
  약 6만 개 타임스탬프에 대해 행 단위로 append하므로 매우 느립니다 —
  `merge_river_data.py`의 pandas 피벗 방식을 쓰세요.
- `1.5connectedcomponent.ipynb` — 관측소들의 **무방향** `networkx` 그래프를 만들고 연결 요소
  (connected component), 즉 독립적인 유역들로 분할해 `set1..setN` 하위 디렉터리와 각각의
  `relationshipset<N>.csv`로 내보냅니다. 노트북 자체가 경고하듯 `nx.connected_components`의 요소
  순서는 실행마다 달라지므로 set 번호는 안정적이지 않습니다 — 세션을 넘어 하드코딩된
  `connectedcom[i]` 인덱스에 절대 의존하지 마세요.
- `3datastructure and export h5 water data.ipynb` — 핵심 결과물. 앞쪽 약 24개 셀은
  **tsl/GRIN의 `AirQuality` 데이터셋 클래스를 그대로 복사**한 것으로(`PandasDataset`,
  `geographical_distance`, `thresholded_gaussian_kernel` 포함) 템플릿으로 쓰였으며, 그 안의
  `from ..utils.utils` import는 단독으로는 해석되지 않습니다. 실제 작업은
  "discharge data read in and apply mask" 셀부터입니다: 한 유역의 discharge를 가져와 마지막 1800
  타임스텝을 잘라내고, 보간 실험용 합성 결측 패턴으로 **AirQuality의 `eval_mask` 블록을 빌려 쓰며**,
  `dist`를 방향성 인접행렬로 만든 뒤 `water.h5`로 저장합니다.

### `water.h5` 구조

tsl의 `AirQuality.load_raw`가 기대하는 형태를 본뜬 세 개의 pandas 키:

| 키 | 내용 |
| --- | --- |
| `discharge` | wide 프레임, DatetimeIndex × 관측소 열 |
| `eval_mask` | 동일한 shape. 1 = 보간 평가를 위해 감춰 둔 실측값 |
| `dist` | 관측소 × 관측소 방향성 인접행렬 (1 = 상류→하류 엣지). 이름과 달리 *거리 행렬이 아님* |

의미상의 차이에 주의하세요: tsl/AirQuality 규약에서 `mask`는 유효한 데이터를, `eval_mask`는 감춰 둔
실측값을 표시하며, `dist`는 보통 가우시안 커널을 통과시킨 지리적 거리를 담습니다. 여기서 `dist`는
그 대신 이진 하천망 인접행렬이므로, 템플릿의 `get_similarity` / `thresholded_gaussian_kernel`을
여기에 적용해서는 안 됩니다.
