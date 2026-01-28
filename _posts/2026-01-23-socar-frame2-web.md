---
layout: post
title: "SOCAR-FRAME2.0 디자인 시스템 개발기(웹)"
subtitle: 라이브러리를 넘어 시스템으로
date: 2026-01-23 00:00:00 +0900
category: fe
background: "/img/2026-01-23-socar-frame2-web/logo.png"
author: arnold
comments: true
tags:
  - design-system
---

<br />

# 목차

1. [개요](#1-개요)
2. [시스템 설계](#2-시스템-설계)
   1. [설계 목적](#21-설계-목적)
   2. [Figma Plugin](#22-Figma-Plugin)
   3. [Figma Code Connect](#23-Figma-Code-Connect)
3. [컴포넌트 설계](#3-컴포넌트-설계)
   1. [설계 목적](#31-설계-목적)
   2. [Hook과 객체](#32-Hook과-객체)
   3. [합성 컴포넌트와 명령형 API](#33-합성-컴포넌트와-명령형-API)
4. [패키지 전략](#4-패키지-전략)
   1. [트리쉐이킹](#41-트리쉐이킹)
5. [AI 활용](#5-AI-활용)
   1. [Instructions](#51-Instructions)
   2. [Figma MCP with LLM](#52-Figma-MCP-with-LLM)
6. [디자인 시스템 구현 과정](#6-디자인-시스템-구현-과정)
7. [후기](#7-후기)

---

<br /><br />

# 1. 개요

안녕하세요 쏘카 개발자 아놀드입니다.

쏘카에서 장기간의 프로젝트인 socar-frame2.0 개발에 참여하며 고민하고 개발하였던 이야기를 해보려 합니다.

> 각 파트별로 할 수 있는 이야기가 많지만 본 글에서는 시스템의 큰 방향과 약간의 세부사항에 대한 이야기를 적어보겠습니다.

쏘카에는 **socar-frame**이라는 기존의 UI 라이브러리가 존재하였습니다.

기존 UI 라이브러리는 디자인 시스템을 표방하였으나, 입사 당시 이미 라이브러리의 R&R이 모호한 상태로 유지보수가 거의 되지 않고 파편화된 UI/UX 개발이 각 서비스별로 이뤄지고 있는 상황이었습니다.

따라서, 사내 디자이너와 FE개발로 이어지는 제품 개발 프로세스에 일관성과 효율성이 떨어지는 상태였고 이로 인해 시스템의 개선 및 추가개발을 진행하여 `SOCAR-FRAME 2.0`의 개발을 진행하는 프로젝트가 시작되었습니다.

# 2. 시스템 설계

## 2.1 설계 목적

기존의 사례가 존재하였기에 최대한 변화에 대응하며 확장성이 높은 상태로 UI 라이브러리를 설계하는 것이 목표였습니다.

그러나 UI 라이브러리만 제공한다면 생산성의 향상은 기존 대비 크지 못할 것이라고 보았고, 당시 팀원들과 자료 수집 및 논의를 통해 확실한 **시스템**으로서의 설계안을 만들기 위해 고민하였습니다.

쏘카는 사내 디자인 툴로 Figma를 활용하고 있었고, 이를 통한 디자인을 개발자가 한땀한땀 구현으로 옮기는 것이 기존 업무 방식이었습니다.

따라서 이 업무 프로세스에 날개를 달아줄 방법들을 위한 고민을 많이 하던 중 몇몇 부분의 최적화가 가능하다고 봤습니다.

`디자이너 -> 개발자` 로 전달되는 프로세스와 `개발자(라이브러리) -> 개발자(서비스)` 로 전달되는 프로세스입니다.

이 과정은 Figma의 지원기능을 백분 활용하였습니다.

## 2.2 Figma Plugin

`디자이너 -> 개발자(서비스)`방향으로 소통하는 과정의 최적화 중 일부는 정적인 에셋의 처리였습니다.

구상은 아래 그림과 같았고 이를 위해 Figma Plugin을 활용했습니다.

![socar-frame2.0](/img/2026-01-23-socar-frame2-web/socar-frame2.png)

기존에는 아이콘·스페이싱 토큰이 슬랙 → 담당 개발자 수동 PR 생성 → 검증 → 배포로 이어져 리드타임이 길었습니다.

이를 줄이기 위해 Figma 내부 플러그인을 도입해 디자이너가 바로 PR을 생성하도록 했고, 현재는 PAT(Personal Access Token) 기반으로 인증을 처리합니다.

운영상 보안·권한 문제는 남아 있어, 별도 인증 서버 도입을 후속 과제로 검토 중입니다.

![socar-frame2.0](/img/2026-01-23-socar-frame2-web/figma-plugin.png)

![icon-pr](/img/2026-01-23-socar-frame2-web/icon-pr.png)

이 외에도 토큰과 같은 foundation 레벨의 요소가 추가될 때는 위와 같은 프로세스를 확장시켜 활용할 수 있을 것으로 보고 있습니다.

## 2.3 Figma Code Connect

`개발자(라이브러리) -> 개발자(서비스)`방향으로 소통하는 과정의 최적화 중 일부는 UI 라이브러리 코드 사용법입니다.

기존 디자인 시스템의 활용은 스토리북을 통해 UI를 확인하고, 라이브러리를 포함한 레포지토리 내에서 코드를 확인하여 실제 서비스에 구현하는 방식이었습니다.
하지만 Figma Code Connect를 잘 활용한다면 이 과정을 간소화할 수 있게 됩니다.

![figma-code-connect-code](/img/2026-01-23-socar-frame2-web/figma-code-connect-code.png)

Figma Code Connect를 적용하면 디자이너가 선택한 노드에 대해 대응되는 UI 라이브러리 코드가 즉시 노출돼 사용 흐름이 단축됩니다.

![figma-code-connect](/img/2026-01-23-socar-frame2-web/figma-code-connect.png)

사진과 같이 특정 UI를 클릭하면 그에 맞는 UI 라이브러리 코드가 나타나게 됩니다.

다만 DatePicker처럼 상태/변형이 많은 컴포넌트는 매핑이 쉽지 않았습니다.
이 과정에서 서로의 설계 방식에 대한 이해가 충분하지 않아 적지 않은 러닝커브가 있었습니다.

디자인 직군은 Figma의 Variant를 활용해 컴포넌트의 노출 여부를 토글하는 방식이 익숙했습니다.
시스템에 있는 컴포넌트를 복사해 Variant를 변경한 뒤 서비스에 적용하는 방식을 주로 사용하셨지만 Slot 형태를 쓰면 복사할 때마다 Slot에 넣을 컴포넌트를 변경해야 하는 번거로움이 있어 자주 쓰지 않는 방식이었습니다.

반면 개발은 합성 컴포넌트 구조에서 어떤 컴포넌트를 하위에 두더라도 유연하게 대응하는 것이 우선순위였기에, 작업 방식의 차이가 존재했습니다.

하지만 합성 컴포넌트 구조에서 이 방식을 그대로 적용하면 Code Connect를 위한 코드와 디자인의 정합성이 떨어진다고 판단했습니다.
그래서 Slot 개념을 활용하기로 합의했고, 이를 통해 확장성을 확보했습니다.

서로 다른 업무 방식의 차이로 인해 현재는 디자인 시스템 영역에서만 이 방식을 사용하고 있습니다.

아래는 특정 Slot 안에 정의된 children이 자유롭게 들어갈 수 있도록 개선한 예시입니다.

아래 사진은 Figma내에서 하위 UI들이 Main Component로 정의되어있습니다.
![datepicker-figma](/img/2026-01-23-socar-frame2-web/datepicker-figma.png)

아래 보라색 텍스트는 이미 Main Component로 정의된 컴포넌트를 Slot 형태로 정의된 children에 넣어 다른 Main Component내에서 유연하게 연결할 수 있게 된 예시입니다.

![datepicker-figma-slot](/img/2026-01-23-socar-frame2-web/datepicker-figma-slot.png)

위 형태를 기반으로 Code Connect를 할 수 있게 되었습니다.
![datepicker-figma-slot-code](/img/2026-01-23-socar-frame2-web/datepicker-figma-slot-code.png)

결과적으로 합성 컴포넌트 형태와 Figma 설계 형태가 더 유사해졌습니다.

# 3. 컴포넌트 설계

## 3.1 설계 목적

컴포넌트 설계에서 가장 중요하게 본 건 재사용성과 독립성이었습니다.

과거의 경험을 바탕으로 UI는 계속 바뀌는데 그때마다 내부 로직까지 흔들리면 유지보수 비용이 너무 커지기 때문에, 최대한 UI와 로직을 분리하는 구조를 고민했습니다.

이 과정에서 오픈소스 라이브러리들의 설계를 많이 참고했지만, 결국 실제 서비스 운영과 유지보수에 유리한 구조를 기준으로 선택했습니다.

## 3.2 Hook과 객체

구체적인 방향은 **UI와 상태가 강하게 얽히는 컴포넌트**와 **그렇지 않은 컴포넌트**를 나누는 것이었습니다.

기존 컴포넌트들이 재활용되기 어려웠던 가장 큰 이유가, UI 정책과 비즈니스 로직이 섞여 있던 경험이었기 때문입니다.

UI/UX 정책과 비즈니스로직을 잘 분리하여 만들어 놓지 않은 경우 동일 정책에 유사한 UI 컴포넌트를 서비스별로 재개발하는 경우가 있었기 때문입니다.

그래서 `“UI가 꼭 가져야 하는 상태인가, 아니면 정책 로직으로 분리 가능한가”`를 기준으로 구조를 나눴습니다.

강결합되는 컴포넌트는 Hook 중심으로 설계했고, 그렇지 않은 컴포넌트는 로직을 객체로 분리한 뒤 Hook을 어댑터로 두는 방식으로 UI를 분리했습니다.

UI와 상태가 강하게 엮이는 대표 사례가 BottomSheet와 Accordion입니다.

![bottomsheet-ui](/img/2026-01-23-socar-frame2-web/bottomsheet-ui.png)

BottomSheet는 `드래그/스냅 포인트/바운드/스크롤 잠금` 같은 상호작용이 UI와 분리되기 어렵고, 상태 전이가 곧 UI 애니메이션이라 Hook 중심 구조가 자연스럽다고 봤습니다.

따라서, 설계 단계에서 디자이너/네이티브와 충분히 논의한 끝에 상태(hidden, tip, half, max)를 정의했습니다.

아래 일부 발췌된 코드를 참고하면 상태가 UI에 강하게 얽혀있다는 것이 어떤 의미인지 파악할 수 있습니다.

```tsx
// BottomSheet의 상태에 해당하는 case가 innerHeight와 얽혀있는 모습
export const resolveTargetY = ({
  half,
  max,
  state,
  tip,
}: {
  half: number
  max: number
  state: BottomSheetState
  tip: number
}) => {
  switch (state) {
    case 'max':
      return window.innerHeight - max
    case 'tip':
      return window.innerHeight - tip
    case 'half':
      return window.innerHeight - half
    case 'hidden':
    default:
      return window.innerHeight
  }
}
  // 생략..

  //이 상태를 활용하여 animate를 시키는 메서드
  const animateToState = (state: BottomSheetState) => {
    const normalized = normalizeState(state)
    const targetY = resolveTargetY({
      half,
      max,
      state: normalized,
      tip,
    })
    animate(bottomSheetY, targetY, SPRING_TRANSITION)
  }
  // 생략..

  // 기타 복잡한 제어들이 상태에 연결될 수 밖에 없는 형태
  const setState = (next: BottomSheetState) => {
    const normalizedNext = normalizeState(next)
    const current = activeStateRef.current
    if (current === normalizedNext) {
      animateToState(normalizedNext)
      return
    }

    activeStateRef.current = normalizedNext

    if (!isControlled) {
      setInternalState(normalizedNext)
    }

    animateToState(normalizedNext)
    onStateChange?.(normalizedNext)
  }

  ...

    const handleDragEnd = () => {
    if (isHandlingDragEndRef.current) return

    ...

    if (!sheetRef.current) return

    const rect = sheetRef.current.getBoundingClientRect()
    const visibleHeight = window.innerHeight - rect.top
    const currentState = activeStateRef.current

    if (currentState === 'hidden') {
      animateToState('hidden')
      return
    }

    if (currentState === 'tip') {
      if (withoutTip) {
        setState('half')
        return
      }

      if (visibleHeight < tip - DRAG_THRESHOLD) {
        emitDragClose()
        return
      }

      if (visibleHeight > tip + DRAG_THRESHOLD) {
        setState('half')
        return
      }

      animateToState('tip')
      return
    }

    if (currentState === 'half') {
      if (visibleHeight < half - DRAG_THRESHOLD) {
        if (isFooterExist || withoutTip) {
          setState('hidden')
          return
        }

        setState('tip')
        return
      }

      if (max > half && visibleHeight > half + DRAG_THRESHOLD) {
        setState('max')
        return
      }

      animateToState('half')
      return
    }

    if (currentState === 'max') {
      const maxTargetY = window.innerHeight - max
      const currentY = bottomSheetY.get()
      const draggedDownDistance = currentY - maxTargetY

      if (max <= half) {
        if (isFooterExist) {
          const shouldHide =
            draggedDownDistance > DRAG_THRESHOLD ||
            visibleHeight < max - DRAG_THRESHOLD

          if (shouldHide) {
            setState('hidden')
            return
          }

          animateToState('max')
          return
        }
      }

      const shouldCollapse =
        draggedDownDistance > DRAG_THRESHOLD || max - half < DRAG_THRESHOLD

      if (shouldCollapse) {
        setState(isFooterExist ? 'hidden' : 'half')
        return
      }

      animateToState('max')
      return
    }

    setState('half')
  }
  // 생략...

```

더 많은 요구사항이 있을 경우 상태머신이나 아래 다른 예제와 같이 별도 객체로 분리하는 것을 고려해볼 수 있었으나, 현재 단계에서는 Hook으로 충분하다고 보았고 이미 서비스들에서 요구하는 많은 케이스들을 감당할 만한 정책이라고 결론냈습니다.

![bottomsheet-hooks](/img/2026-01-23-socar-frame2-web/bottomsheet-hooks.png)

전체 구조는 사진과 같이 `useBottomSheet.tsx`에서 다양한 훅들을 orchestration하여 활용하였습니다.

반대로 DatePicker나 Pattern(Carousel)은 UI보다 정책 로직이 훨씬 복잡하다고 봤습니다.

![datepicker-ui](/img/2026-01-23-socar-frame2-web/datepicker-ui.png)

예를 들어 DatePicker는 사용자에게는 start/end 선택 UI만 보입니다. 하지만 내부적으로는 날짜 계산, 라벨 처리, 비활성 정책 등 UI와 직접 연결되지 않는 로직이 많았습니다.

그래서 이런 로직은 아래와 같이 객체로 분리하였습니다.

![datepicker-architecture](/img/2026-01-23-socar-frame2-web/datepicker-architecture.png)

```ts
// core/DateManager.ts
export class DateManager {
  constructor(today = new Date(), options: DateManagerOptions = {}) {
    this.labelService = new LabelService({
      generateId: (date) => this.generateId(date),
      getHolidays: () => this.holidays,
      getToday: () => this.today,
      isSameDate: (a, b) => this.isSameDate(a, b),
    });
    this.disablePolicy = new DisablePolicy({
      compareDates: (d1, d2) => this.compareDates(d1, d2),
      getDisablePast: () => this.disablePast,
      getHolidays: () => this.holidays,
      getManualDisabledDates: () => this.manualDisabledDates,
      getSelectableEnd: () => this.selectableEnd,
      getSelectableStart: () => this.selectableStart,
      getToday: () => this.today,
      isSameDate: (a, b) => this.isSameDate(a, b),
    });
    this.gridBuilder = new CalendarGridBuilder({
      compareDates: (d1, d2) => this.compareDates(d1, d2),
      disablePolicy: this.disablePolicy,
      labelService: this.labelService,
      todayProvider: () => this.today,
    });
    this.selectionService = new SelectionService({
      applyMaxRangeDays: (target) => this.applyMaxRangeDays(target),
      compareDates: (d1, d2) => this.compareDates(d1, d2),
      disablePolicy: this.disablePolicy,
      emitChange: () => this.emitChange(),
      generateId: (date) => this.generateId(date),
      getDateMap: () => this.dateMap,
      getNodeById: (id) => this.dateMap.get(id),
      isFlushApply: () => this.isFlushApply,
      labelService: this.labelService,
      resetAllSelections: () => this.resetAllSelections(),
      setFlushApply: (value) => {
        this.isFlushApply = value;
      },
      updateDateObject: (id, updates) => this.updateDateObject(id, updates),
      withDuplicate: options.withDuplicate ?? false,
    });
  }

  // 생략...

  select(date: Date) {
    this.selectionService?.select(date);
  }
}
```

중간에 Hook으로 Adapter를 두어 React의 라이프사이클을 안정적으로 따르게 하였습니다.

```ts
// hooks/useDateManager.ts
export const useDateManager = (
  manager: DateManager,
  initialSetting?: () => void,
) => {
  const [calendarMap, setCalendarMap] = useState<CalendarMap | null>(null);

  useEffect(() => {
    setCalendarMap(
      new Map(manager.getDateGridFromMap(manager.dateMap, manager.dateRecords)),
    );

    const unsubscribe = manager.subscribe(() => {
      setCalendarMap(
        new Map(
          manager.getDateGridFromMap(manager.dateMap, manager.dateRecords),
        ),
      );
    });

    initialSetting?.();
    return () => unsubscribe();
  }, [manager]);

  const handleSelect = (date: Date) => manager.select(date);

  return { calendarMap, handleSelect };
};
```

그리고 그 결과를 UI로 emit하는 역할만 하도록 구성했습니다.

```tsx
// DatePicker 사용부 (stories 예시)
const managerRef = useRef(new DateManager(undefined, { withDuplicate: true }));
const { calendarMap, handleSelect } = useDateManager(managerRef.current, () => {
  managerRef.current.setMonthCount(0, 2);
  managerRef.current.setCustomLabel([
    { date: new Date("2025-12-25"), labelConfig: { label: "christmas" } },
  ]);
});

return (
  <>
    {Array.from(calendarMap?.entries() ?? []).map(([key, weeks]) => (
      <WeekGroup key={key} weeks={weeks} handleSelect={handleSelect} />
    ))}
  </>
);
```

물론 이런 구조는 러닝커브와 복잡도를 높이는 단점이 있습니다.

하지만 테스트 가능성과 정책 재사용성 측면에서는 그 비용을 상쇄할 수 있다고 봤고, 실제로 유지보수 관점에서도 더 안정적인 구조가 된다고 느꼈습니다.

## 3.3 합성컴포넌트와 명령형 API

### 합성형 컴포넌트

많은 UI 라이브러리에서 사용하는 JSX 합성 패턴을 그대로 가져가면서, 최대한 자율적으로 조합할 수 있게 했습니다.
특히 data attribute를 적절히 배치해 pseudo class로 커스텀 스타일링이 가능하도록 설계했습니다.

```tsx
<Tab
  className="
    [&_[data-slot=tab-container]]:tw-bg-gray-50
    [&_[data-slot=tab-indicator]]:tw-bg-red-500
    [&_[data-slot=tabs-content-slide]]:tw-rounded-radius-300
  "
>
  <Tab.Header>
    <Tab.HeaderItem>택시</Tab.HeaderItem>
    <Tab.HeaderItem>카페</Tab.HeaderItem>
  </Tab.Header>

  <Tab.Indicator />

  <Tab.Content>
    <Tab.ContentItem>콘텐츠 A</Tab.ContentItem>
    <Tab.ContentItem>콘텐츠 B</Tab.ContentItem>
  </Tab.Content>
</Tab>
```

라이브러리 1차 개발이 끝나기도 전에 몇몇 서비스에서 선반영이 진행됐기 때문에, 중간중간 패치로 대응하기보다는 개발자가 스스로 조정할 수 있는 여지를 넓혀주는 쪽이 더 적합하다고 봤습니다.
그래서 data attribute 기반의 스타일링 접근을 적극적으로 권장했습니다.

### 명령형 API

Alert는 상태나 단계에 따라 다른 UI/로직이 필요한 경우가 많았습니다.
단계가 늘어날수록 상태 분기와 콜백 흐름이 꼬이고, 코드만 봐서는 전체 흐름을 파악하기 어려워지는 문제가 있었기 때문입니다.

그래서 Alert.open을 통해 Alert를 열고, 버튼 클릭 결과를 Promise로 받는 패턴을 제공했습니다.
이렇게 하면 UI 내부에서 상태를 계속 갱신하기보다, 호출부에서 결과에 따라 분기하는 흐름을 만들 수 있습니다.

```tsx
import {
  ActionButton,
  Alert,
  IconExclamationCircleFill,
  type OnAction,
} from "@socar-inc/socar-frame-components";

const BasicAlert = ({ onAction }: { onAction: OnAction }) => {
  return (
    <Alert withDim>
      <Alert.GraphicSlot graphicHeight={120}>
        <IconExclamationCircleFill
          className="tw-fill-status-caution-regular"
          size={48}
        />
      </Alert.GraphicSlot>
      <Alert.Title>알럿 타이틀</Alert.Title>
      <Alert.Body>상세 설명이 들어갑니다.</Alert.Body>
      <Alert.LinkButton
        label="버튼명"
        onClick={() => onAction("link")}
        size="small"
        type="button"
        underline
        variant="primary"
      />
      <Alert.ButtonSlot>
        <ActionButton
          className="tw-w-full"
          label="확인"
          onClick={() => onAction("confirm")}
          size="medium"
          type="button"
          variant="primary"
        />
        <ActionButton
          className="tw-w-full"
          label="취소"
          onClick={() => onAction("cancel")}
          size="medium"
          type="button"
          variant="secondary"
        />
      </Alert.ButtonSlot>
    </Alert>
  );
};

const handleConfirm = async () => {
  console.log("확인 클릭됨");
};

const handleCancel = async () => {
  console.log("취소 클릭됨");
};

const handleLink = async () => {
  console.log("링크 클릭됨");
};

export const DefaultAlertExample = () => {
  const handleClick = async () => {
    const result = await Alert.open((onAction) => (
      <BasicAlert onAction={onAction} />
    ));

    if (result === "confirm") await handleConfirm();

    if (result === "cancel") await handleCancel();

    if (result === "link") await handleLink();
  };

  return (
    <>
      <ActionButton
        label="Default Alert 열기"
        onClick={handleClick}
        size="medium"
        type="button"
        variant="primary"
      />
    </>
  );
};
```

물론 단계가 많아지면 여전히 콜백/분기 코드가 길어질 수 있습니다.

하지만 상태를 계속 바꾸면서 UI를 조작하는 방식보다, UI를 열고 결과에 따라 처리하는 흐름이 더 명확하다고 느꼈습니다.

# 4. 패키지 전략

## 4.1 트리쉐이킹

다른 UI 라이브러리들과 유사하게 **데드 코드를 제거**할 수 있는 환경을 제공하였습니다.

초기에는 tsup 기반 번들링으로 개발했는데, 모듈 구조가 유지되지 않아 트리쉐이킹에 불리한 점이 있었습니다.

그래서 components 패키지는 rollup으로 전환했고, preserveModules / preserveModulesRoot를 통해 서비스 번들러가 트리쉐이킹을 위한 폴더 Graph 생성에 유리한 구조로 변경하였습니다.

rollup으로 변경한 또 다른 이유는 SSR에 대한 대응이 있었습니다.

쏘카는 현재 next 13~15 버전들이 혼재되어 서비스에서 활용되고 있었고, 13버전의 기본 CSR 지원 형태와 다르게 14~15로 변경되며 `use client`의 상단부 선언에 대한 니즈가 발생하였습니다.

이 부분을 tsup으로 처리하는 것보다 rollup의 banner 옵션으로 각 출력 모듈 파일 상단에 'use client'를 주입하는 방식이 훨씬 단순하다고 봤습니다.

```ts
...
const outputBase = {
  dir: 'dist/src',
  preserveModules: true,
  preserveModulesRoot: 'src',
}
...

export default defineConfig([
  {
    external,
    input,
    output: [
      {
        ...outputBase,
        banner: useClientBanner,
        entryFileNames: '[name].js',
        format: 'esm',
      },
      {
        ...outputBase,
        banner: useClientBanner,
        entryFileNames: '[name].cjs',
        exports: 'named',
        format: 'cjs',
      },
    ],
    plugins: [
      esbuild({
        jsx: 'automatic',
        minify: false,
        target: 'es2018',
        tsconfig: 'tsconfig.json',
      }),
      json(),
    ],
    treeshake: true,
  },
  ...
)]
```

이렇게 useClientBanner와 preserveModules를 적용한 뒤, 특정 서비스 기준으로 번들 전후 비교를 진행했습니다.

여기서 external은 rollup 번들 단계에서 공통 의존성을 분리해 서비스 번들러의 중복 청크 생성을 줄이기 위한 설정입니다.

결과는 아래와 같이 유의미한 성과를 가져왔습니다.

![tree-shake](/img/2026-01-23-socar-frame2-web/tree-shake.png)

- 공통 static/chunk의 용량을 **1.45mb → 567.07kb** 로 **약 61%의 감소**
- first load js는 **373kb → 248kb**로 **33%의 청크 번들 감소**
- external의 효과로 **pages_app-xxx.js로 생기는 큰 청크 제거**
- preserveModules를 통해 **트리쉐이킹 친화적 구조 생성**
- 실제 pages와 연결되어 있는 chunk 파일 확인 시 **필요한 컴포넌트만 node_modules 안에 빌드**된 것 확인

# 5. AI 활용

## 5.1 Instructions

LLM을 잘 활용하려면 사전에 정리된 내용을 명확히 전달하는 것이 중요하다고 생각했습니다.
이 글에서 언급하는 Instructions는 LLM에게 사전 정의된, 의도된 결과의 정확도를 높이기 위한 사전 제공되는 지시 문서로서의 개념입니다.
이 Instructions에는 [AGENTS.md](https://agents.md/)와 [llms.txt](https://llmstxt.org/)의 개념을 활용했습니다.
AI를 실무에 쓰려면 “문서가 있다” 수준을 넘어 UI 라이브러리 사용법, Figma 설계 규칙, 예외 처리 기준까지 한 번에 전달돼야 한다고 봤습니다.

그래서 내부 공식 문서 사이트를 통해 정적인 가이드를 제공했고, LLM이 빠르게 맥락을 잡을 수 있도록 `llms.txt`를 운영했습니다.

`llms.txt`는 전체 덤프(`llms-full.txt`), 컴포넌트 인덱스(`index.txt`)로 연결되어 있어, LLM이 필요한 정보를 단계적으로 찾을 수 있게 구성됩니다.
즉, 문서를 “사람이 읽기 쉬운 형태”로만 두지 않고, LLM이 바로 소비할 수 있는 구조로 별도 정리해 둔 셈입니다.

`AGENTS.md`에 이 모든 것을 기재할 수도 있으나 각 파일 별 영역을 나누기 위해 이 개념을 나눠 진행하였습니다.

그리하여 실제 서비스 레포에서는 `AGENTS.md`를 참고 해 라이브러리 사용 규칙과 원칙을 명확히 전달했습니다.

이 작업은 글 초반에 말한 `디자이너 → 개발자(라이브러리) → 개발자(서비스)` 흐름에서 중간 커뮤니케이션 비용을 줄이기 위한 핵심 축이라고 봤습니다.

![agent-template](/img/2026-01-23-socar-frame2-web/agent-template.png)

디자이너 의도 파악 비용, 개발자 간 UI 정책 확인 비용, PM의 정책 검증 비용을 줄이려면 사전에 합의된 규칙이 문서로 고정되어야 했습니다.

합의된 규칙을 명세로 정리해 활용한 뒤, AI로 테스트 코드를 작성하는 과정이 훨씬 수월해지는 걸 경험했습니다.
그 과정에서 규칙의 중요성을 한층 더 체감했습니다.

![agent](/img/2026-01-23-socar-frame2-web/agent.png)

그래서 UI 라이브러리를 “단순 라이브러리”가 아니라 시스템으로 작동하게 만들기 위한 기반 문서로 `llms.txt`와 `AGENTS.md`를 함께 운영하게 되었습니다.

## 5.2 Figma MCP with LLM

사내 LLM 서비스에 Figma MCP를 접목하면, 에디터 내에서 특정 노드 ID를 기준으로 `디자이너 → 개발자(서비스)` 흐름을 더 직접적으로 만들 수 있습니다.

즉, 디자이너가 그린 결과물을 개발자가 에디터 안에서 바로 코드로 확인하거나 수정하는 방식으로 연결될 수 있습니다. 다만 이 과정에서 결과 품질은 Instruction과 Figma 설계 방식에 매우 민감하다는 걸 확인했습니다.
같은 디자인이라도 Instruction에 어떤 지시가 들어가 있는지, Figma 안에서 슬롯/상태가 어떻게 정의되어 있는지, hidden 노드가 어떤 방식으로 처리되어 있는지에 따라 결과가 크게 달라졌습니다.

| Instruction 개선 전 결과                                   | Instruction 개선 후 결과                                     |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| ![ui-first](/img/2026-01-23-socar-frame2-web/ui-first.png) | ![ui-second](/img/2026-01-23-socar-frame2-web/ui-second.png) |

그래서 이 부분은 단발성으로 끝나는 작업이 아니라, 사례를 계속 쌓고 Instruction과 Figma 설계 규칙을 지속적으로 보완해 나가야 하는 영역이라고 봤습니다.

또한 개발자의 세세한 리뷰를 반영하거나 비즈니스 로직 반영 기준을 정책화하는 것을 추후 개선점으로 가져갈 예정입니다.

# 6. 디자인 시스템 구현 과정

디자인 시스템 개발은 단기간에 끝나는 일이 아니라, 장기 프로젝트로 진행되었습니다.
파트별로 담당 팀/인원이 배정되었고, 그때그때 우선순위가 바뀌는 일도 많았습니다. 다만 큰 흐름은 `정기 회의 → UI/UX 정책 합의 → 구현 → 검증` 구조를 유지했습니다.

먼저 어떤 UI를 먼저 내릴지 우선순위를 정했고, N차 개발 마일스톤을 나눠서 진행했습니다.
이 과정에서 **“구현이 완료된 컴포넌트는 최대한 빠르게 서비스에 반영해야 한다”**는 요구가 있었기 때문에, 일부 컴포넌트는 `부분 적용 → 피드백 → 수정`의 흐름으로 운영되기도 했습니다.

그 과정에서 OS별 구현 방식 차이(모션, 스크롤, 터치/제스처 등)나 구조적으로 피할 수 없는 한계도 확인했습니다.
그리고 비개발 직군인 디자이너와 개발 직군 사이에서 Code Connect 매핑 규칙을 맞춰나가는 과정이 생각보다 큰 챌린지였습니다.
컴포넌트 슬롯 구조나 상태 정의가 Figma 설계와 맞아야 했기 때문에, 단순히 “연결”이 아니라 서로의 설계 방식 자체를 합의하는 과정이 필요했습니다.
이런 상황 속에서 각 직군의 컨텍스트를 맞추고 시스템의 일관성을 유지하기 위해, 큰 틀의 정책은 합의한 뒤 담당 개발자가 구현하고 QC를 거쳐 배포 전략을 결정하는 구조를 유지했습니다.

# 7. 후기

각 파트별로 정말 많은 이야기를 나눌 수 있는 주제들이지만, 이 글에서는 방향과 과정 위주로 공유하려 했습니다.
세부 구현까지 모두 담기엔 부족하지만, 그만큼 넓게 봤던 고민과 경험을 기록해 두고 싶었습니다.

개인적으로 가장 큰 수확은 컴포넌트 설계 방식과 번들 구조에 대해 깊게 고민해 본 경험이었습니다.
특히 preserveModules와 트리쉐이킹을 실제 서비스 번들에서 확인해보는 과정은, 단순히 “번들 크기 줄이기”를 넘어서 라이브러리 구조가 소비 환경에 어떤 영향을 주는지 체감하게 해주었습니다.
또한 BottomSheet처럼 UI와 상태가 강결합된 컴포넌트와, DatePicker처럼 로직을 분리해야 하는 컴포넌트를 나누어 설계하면서 재사용성과 테스트 가능성에 대한 관점을 다시 정리할 수 있었습니다.

LLM 친화적인 환경을 만들기 위한 고민도 의미 있는 경험이었습니다.
AGENTS.md와 llms.txt 같은 instruction 체계를 어떻게 구성해야 “문서가 아닌 실질적인 가이드”가 되는지, 그리고 Figma 설계 규칙과 어떻게 맞물려야 하는지 시행착오를 겪었습니다.
이 과정에서 **“기술보다 먼저 합의되어야 하는 규칙”**이 얼마나 중요한지도 체감했습니다.

아직 `SOCAR-FRAME 2.0`에는 상위 버전 대응을 위한 의존 패키지 업그레이드, AI·Figma 플러그인 보강 등 남은 과제가 있습니다.
그럼에도 1차 UI 개발과 시스템 기반을 마련한 것은 큰 진전이었고, SOCAR의 클라이언트 개발이 어떤 방향으로 디자인 시스템을 고민했는지 공유할 수 있어 의미 있는 기록이라고 생각합니다.

이번 단계에서는 구조를 만드는 데 집중했기 때문에 정량 지표는 아직 부족합니다.
다음 단계에서는 토큰 반영 리드타임, 핸드오프 질의량, 번들 영향 등을 측정해 시스템의 효과를 수치로 검증할 예정입니다.

마지막으로, 이 과정에 함께해 준 동료들에게 진심으로 감사드립니다.
여러 동료들의 도움을 통해 이 단계까지 올 수 있게 된 것 같습니다.
이 글을 빌어 함께 진행했던, 진행하고 있는 동료들에게 다시 한 번 감사 인사를 전합니다.

## 7.1 사내 공식 문서

관심 있으신 분들의 참고를 위해 사내 [공식문서](https://socarframe.socar.me/)를 공유드립니다.

현재 라이브러리는 private 상태입니다.
