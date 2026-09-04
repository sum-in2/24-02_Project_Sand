# Project Sand

실크로드를 배경으로 한 무역 상인 시뮬레이션 게임입니다. 플레이어는 사마르칸트, 메르브, 니샤푸르, 바그다드, 다마스쿠스, 예루살렘, 알렉산드리아 등 실크로드의 여러 도시를 오가며 상인들과 물품을 사고팔아 부를 축적합니다.

<!-- 데모 GIF/스크린샷 자리 -->

CSV 한 줄이 곧 게임 데이터가 되도록, 국가·난이도 조합을 키로 삼아 아이템을 파싱합니다.

```csharp
var key = (country, difficulty);
if (!res.ContainsKey(key))
    res[key] = new List<Item>();

res[key].Add(new Item(sprite, element[1],
    int.Parse(element[3]), int.Parse(element[4])));
```

## 게임 소개

- 지도(Map) 위에서 도시를 선택해 이동합니다.
- 각 도시는 난이도(Low / Mid / High)에 따라 취급하는 아이템과 상인 구성이 다릅니다.
- 도시에 도착하면 해당 도시의 상인 목록을 확인하고, 교환(Exchange) UI를 통해 아이템을 거래할 수 있습니다.
- 상단바에서 신뢰도, 소지금, 인원 수, 전투력, 낙타 수 등 플레이어 상태를 확인할 수 있습니다.
- 도시별로 고가 아이템 목록과 가격 변동(상승/하락)을 확인할 수 있습니다.

## 개발 환경

- **엔진**: Unity `2022.3.62f3` (LTS)
- **렌더 파이프라인**: Universal Render Pipeline (URP)
- **주요 패키지**
  - Cinemachine
  - Input System
  - TextMeshPro
  - Timeline
  - Visual Scripting
  - 2D Sprite
  - Test Framework

> 네트워킹 관련 패키지는 포함되어 있지 않으며, 싱글플레이어 프로젝트입니다.

## 핵심 구현

### 1. 스택 기반 UI 매니저

인벤토리, 거래창, 도시 정보 등 여러 UI 패널이 겹쳐 열릴 수 있어, `Stack<GameObject>`로 열림 순서를 관리하고 각 패널의 `Canvas.sortingOrder`를 스택 깊이에 맞춰 자동으로 조정했습니다. 닫을 때는 스택에서 pop하며, 거래창이 닫히면 진행 중이던 교환을 취소하도록 훅을 걸어뒀습니다.

```csharp
public void OpenUI(GameObject uiPanel)
{
    uiPanel.SetActive(true);
    uiPanel.transform.parent.GetComponent<Canvas>()
        .sortingOrder = uiStack.Count + 1;
    uiStack.Push(uiPanel);
}
```

### 2. CSV 기반 데이터베이스 싱글톤

아이템·상인·국가 데이터를 하드코딩 대신 `Resources/DB`의 CSV로 관리합니다. `(Country, Difficulty)` 튜플을 키로 쓰는 딕셔너리에 담아, 기획 데이터가 바뀌어도 코드 수정 없이 CSV만 교체하면 반영되도록 했습니다.

```csharp
public Dictionary<(Country, Difficulty), List<Item>> LoadCountryCSV(string fileName)
{
    var res = new Dictionary<(Country, Difficulty), List<Item>>();
    // country, difficulty를 파싱해 튜플 키로 그룹핑
    ...
    return res;
}
```

### 3. 거래 밸런스 저울 시각화

교환 UI에서 플레이어가 내놓은 아이템 가치와 상인이 받는 아이템 가치를 비교해, 기울기 정도에 따라 저울 스프라이트를 5단계로 전환합니다. 수치 비교를 텍스트가 아닌 직관적인 시각 피드백으로 바꾼 부분입니다.

```csharp
int difference = player - trader;
if (difference > maxValue / 2) imageObject.sprite = scaleImages[0];
else if (difference > 0) imageObject.sprite = scaleImages[1];
else if (difference == 0) imageObject.sprite = scaleImages[2];
else imageObject.sprite = scaleImages[3];
```

## 프로젝트 구조

```
Assets/
├─ Resources/        # CSV 데이터, 프리팹, 이미지 등 런타임 로드 리소스
│  ├─ DB/            # 아이템/상인/국가 CSV 데이터
│  ├─ Exchange/
│  ├─ FBX/
│  ├─ ItemImage/
│  ├─ Map/
│  ├─ Prefabs/
│  └─ Public/
├─ Scenes/
│  ├─ map.unity       # 메인 지도 씬
│  └─ Country.unity    # 도시 상세 씬
├─ Scripts/
│  ├─ Country/        # 지도/도시 관련 스크립트
│  ├─ DB/             # CSV 기반 데이터베이스 싱글톤
│  ├─ Serialize/      # 아이템/상인 데이터 클래스
│  ├─ UI/             # UI 매니저 및 각종 패널
│  └─ Test/           # 개발용 테스트 스크립트
├─ Settings/          # URP 설정
├─ StarterAssets/      # 에셋 스토어 기본 제공 리소스
└─ TextMesh Pro/
```

## 주요 시스템

### 데이터 관리
- `DataManager` — CSV 데이터 로딩, `Country` 열거형(도시 목록) 정의
- `DB/CountryDB`, `DB/ItemDB`, `DB/TraderDB` — 국가/난이도별 아이템·상인 데이터를 관리하는 싱글톤 데이터베이스

### 플레이어
- `PlayerInfo` — 현재 위치, 소지금, 레벨, 경험치, 병력, 낙타 수 등 플레이어 상태
- `Inventory` — 플레이어 인벤토리 및 소지 아이템/소지금 관리 (싱글톤)

### 지도 및 도시
- `Country/Countries`, `Country/CountryInMap`, `Country/CountryBtn` — 지도 상 도시 아이콘 표시 및 클릭 처리
- `Country/CountryItems`, `Country/HighPriceList` — 도시별 판매 아이템 및 고가 아이템 표시

### 거래(Exchange)
- `UI/ExchangeUI`, `ExchangePanel`, `ExchangeInfo`, `ExchangeButton` — 플레이어-상인 간 아이템 교환 UI
- `TraderInCountry`, `TraderInfo` — 도시 내 상인 목록 및 개별 상인 정보
- `ChangePriceObject` — 아이템 가격 변동 표시

### UI
- `UI/UIManager` — 전체 UI 패널 참조 및 열림 상태(스택) 관리 (싱글톤)
- `UI/InventoryUI`, `Slot`, `ItemInfoPanel` — 인벤토리 슬롯 및 아이템 정보 표시
- `UI/TopbarUI` — 상단 상태바
- `UI/CountryInfo` — 도시 정보 패널, 가격/난이도 관련 정의

## 시작하기

1. Unity Hub에서 Unity `2022.3.62f3` 버전을 설치합니다.
2. 이 저장소를 클론한 뒤 Unity Hub의 `Add project from disk`로 프로젝트를 엽니다.
3. `Assets/Scenes/map.unity` 씬을 열고 실행합니다.

```bash
git clone https://github.com/sum-in2/24-02_Project_Sand.git
```

## TODO

- [ ] `PlayerInfo` 등 일부 스크립트에 남아있는 TODO 정리
- [ ] 세이브/로드 시스템 추가
- [ ] 게임 디자인 문서(GDD) 정리
