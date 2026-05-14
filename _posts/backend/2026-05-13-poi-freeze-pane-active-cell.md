---
title: "POI freeze pane 과 활성 셀의 함정"
description: "Excel 자동 생성 파일이 매번 자체 복구를 띄운 이유"
date: 2026-05-13 09:45:00 +0900
categories: [backend]
tags: [apache-poi, excel, freeze-pane, xlsx, java]
image:
  path: /assets/img/posts/poi-freeze-pane-active-cell/cover.jpg
  alt: Excel freeze pane 과 활성 셀 위치를 시트 구조로 표현한 기술 커버 이미지
---

> **Excel 이 자기 자신을 "복구" 했습니다.** 코드로 만들어 내려준 xlsx 인데, 사용자가 열면 "에러가 있어 파일을 복구했다"는 다이얼로그가 뜨고, 복구가 끝나면 화면은 마지막 컬럼 쪽으로 잘 가 있는데 셀 커서는 엉뚱한 `C1` 에 박혀 있었습니다.

월별 가입자 수를 시트 한 장에 펼쳐 보여주는 Excel 을 Apache POI 5.3.0 으로 생성하고 있었습니다. 시트는 왼쪽 두 컬럼·상단 네 행이 freeze pane 으로 고정되어 있고, 사용자는 파일을 열자마자 "가장 최근 데이터가 입력된 컬럼" 으로 시선이 가야 했죠. 그래서 viewport 와 활성 셀을 코드로 지정하는 메서드가 들어가 있었습니다.

## 화면은 갔는데, 커서가 안 갔다

처음 코드는 이랬습니다.

```java
public void setActiveCellToLastSumRow(int colIndex) {
    CellAddress cellAddress = new CellAddress(1, colIndex);

    int sheetIndex = workbook.getSheetIndex(sheet);
    workbook.setActiveSheet(sheetIndex);
    workbook.setSelectedTab(sheetIndex);
    sheet.setSelected(true);
    sheet.setActiveCell(cellAddress);
    sheet.showInPane(0, Math.max(DATE_COL_START, colIndex));
}
```

의도는 단순합니다 — 시트를 활성화하고, 커서를 `(1, colIndex)` 에 두고, 보이는 영역을 마지막 데이터 컬럼 쪽으로 옮긴다. POI 가 내놓는 API 만 봤을 때는 이 조합이 자연스러운 답처럼 보였습니다.

그런데 실제로 열어보면 결과는 셋이었습니다.

1. Excel 이 "파일에 문제가 있어 복구했다" 는 다이얼로그를 띄우고
2. 복구가 끝나면 화면은 마지막 컬럼 쪽으로 잘 가 있고
3. 셀 커서는 `C1` 에 있다

특히 1번이 결정적이었습니다. **Excel 이 복구를 띄웠다는 건, 우리가 만든 xlsx 가 "Excel 이 유효하다고 보지 않는 상태"** 라는 뜻이거든요. POI 가 시키는 대로 잘 했는데, 결과물은 Excel 입장에서 깨진 파일이었던 셈입니다.

## `showInPane` 이 남긴 흔적

xlsx 는 zip 으로 묶인 xml 묶음입니다. `xl/worksheets/sheet1.xml` 을 열어보면, 시트의 시야 상태는 `<sheetView>` 한 노드 안에 들어 있고, freeze pane 이 걸린 시트는 그 안에 `<pane>` 과 `<selection>` 이 함께 살아야 합니다.

- `<pane>` — 어느 pane 이 활성이고 viewport 의 왼쪽 위가 어느 셀이냐 (`activePane`, `topLeftCell`)
- `<selection>` — 각 pane 별 현재 선택이 무엇이냐 (`pane`, `activeCell`, `sqref`)

문제는, freeze pane 이 걸린 상태에서 `setActiveCell(...)` 만 부르면 POI 가 selection 만 건드리지 pane 의 `activePane` / `topLeftCell` 까지 같이 맞춰주지는 않는다는 점입니다. 거기에 `showInPane(...)` 가 한 번 더 들어가면서 sheetView 안에 **일관성이 깨진 pane/selection 조합** 이 만들어졌고, Excel 은 이걸 수상한 sheetView 로 판정해 자체 복구를 돌렸습니다. 복구되는 과정에서 selection 이 기본값으로 되돌아가니까, 화면은 갔는데 커서는 `C1` 으로 돌아온 거고요.

요약하면 원인은 한 줄입니다. **freeze pane 이 걸린 시트의 sheetView 는 POI 고수준 API 만으로는 일관성 있게 만들 수 없다.**

## pane 과 selection 을 직접 그려주기

해법은 sheetView 의 pane / selection 노드를 XMLBeans 레벨에서 직접 만지는 것이었습니다. `showInPane` 호출은 빼고, freeze pane 인자는 상수화한 다음, `CTSheetView` → `CTPane` → `CTSelection` 을 직접 세팅했습니다.

```java
sheet.createFreezePane(FREEZE_COL_SPLIT, FREEZE_ROW_SPLIT);

CTSheetView sheetView = sheet.getCTWorksheet()
        .getSheetViews().getSheetViewArray(0);
CTPane pane = sheetView.isSetPane()
        ? sheetView.getPane() : sheetView.addNewPane();
pane.setActivePane(STPane.TOP_RIGHT);
pane.setTopLeftCell(new CellAddress(
        FREEZE_ROW_SPLIT,
        Math.max(DATE_COL_START, colIndex)
).formatAsString());

while (sheetView.sizeOfSelectionArray() > 0) {
    sheetView.removeSelection(0);
}
CTSelection selection = sheetView.addNewSelection();
selection.setPane(STPane.TOP_RIGHT);
selection.setActiveCell(activeRef);
selection.setSqref(Collections.singletonList(activeRef));
```

이 코드에서 결정한 세 가지.

- **`showInPane` 제거**. viewport 의 위치는 `pane.setTopLeftCell(...)` 으로 명시합니다.
- **`activePane` 지정**. freeze 가 위·왼쪽으로 걸려 있으니 데이터 영역은 `TOP_RIGHT` 이고, selection 도 같은 pane 에 매답니다.
- **기존 selection 을 비우고 새로 추가**. POI 가 만들어둔 기본 selection 이 남아 있으면 이번에도 또 어긋납니다.

이렇게 만든 파일은 Excel 이 복구를 띄우지 않고 그대로 열렸고, 화면은 마지막 데이터 컬럼 쪽으로, 커서는 의도한 셀에 정확히 위치했습니다.

## 가져갈 두 가지

**Excel 이 "복구했다" 고 말하면, 우리가 만든 xml 이 깨졌다는 신호입니다.** Excel 의 자동 복구는 친절한 행동처럼 보이지만, 만든 사람 입장에선 가장 강한 디버깅 단서입니다. xlsx 의 압축을 풀어 `xl/worksheets/sheet1.xml` 의 `<sheetView>` 를 눈으로 한 번 보면, POI 고수준 API 가 만든 결과가 정말 의도대로인지 1분 안에 확인됩니다.

**POI 의 `setActiveCell` / `showInPane` 은 freeze pane 환경의 sheetView 를 책임지지 않습니다.** freeze pane 이 걸려 있다면 pane (`activePane`, `topLeftCell`) 과 selection (`pane`, `activeCell`, `sqref`) 을 한 묶음으로 같이 채워야 일관성이 맞춰집니다. 둘 중 하나만 손대면 Excel 이 "복구" 를 합니다 — 그 복구가 우리 의도를 지워버리고요.
