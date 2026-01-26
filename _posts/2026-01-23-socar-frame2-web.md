---
layout: post
title: "SOCAR-FRAME2.0 디자인시스템 개발기(웹)"
subtitle: 라이브러리를 넘어 시스템으로
date: 2026-01-23 00:00:00 +0900
category: fe
background: "/img/2025-06-11-monorepo-ci-cd-pipeline/matrix.png"
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
   3. [Figma Code Connect](#22-Figma-Code-Connect)
3. [컴포넌트 설계](#3-컴포넌트-설계)
   1. [설계 목적](#31-설계-목적)
   2. [Hook과 객체](#32-Hook과-객체)
   3. [합성컴포넌트와 선언형 메서드](#33-캐시를-활용한-빌드-최적화)
4. [패키지-전략](#4-패키지-전략)
   1. [트리쉐이킹](#41-트리쉐이킹)
5. [AI활용](#5-AI활용)
   1 [Instructions](#52-instructions)
   2 [Figma MCP with LLM](#52-Figma-MCP-with-LLM)
6. [업무진행](#7-업무진행)
7. [후기](#8-후기)

---

<br /><br />

# 1. 개요

안녕하세요 쏘카 개발자 아놀드입니다.

쏘카에서 장기간의 프로젝트인 socar-frame2.0 개발에 참여하며 고민하고 개발하였던 이야기를 해보려 합니다.

> 각 파트별로 할 수 있는 이야기가 많지만 본 글에서는 시스템의 큰 방향과 약간의 세부사항에 대한 이야기를 적어보겠습니다.

쏘카에는 **socar-frame**이라는 기존의 UI 라이브러리가 존재하였습니다.

기존 UI라이브러리는 디자인시스템을 표방하였으나, 입사 당시 이미 라이브러리의 R&R이 모호한 상태로 유지보수가 거의 되지 않고 파편화된 UI/UX개발이 각 서비스별로 이뤄지고 있는 상황이었습니다.

따라서, 사내 디자이너와 FE개발로 이어지는 제품 개발 프로세스에 일관성과 효율성이 떨어지는 상태였고 이로 인해 시스템의 개선 및 추가개발을 진행하여 `SOCAR-FRAME 2.0`의 개발을 진행하는 프로젝트가 시작되었습니다.

# 2. 시스템 설계

## 2.1 설계 목적

기존의 사례가 존재하였기에 최대한 변화에 대응되며 확장성이 높은 상태로 UI라이브러리를 설계하는 것이 목표였습니다.

그러나 UI라이브러리만 제공한다면 생산성의 향상은 기존 대비 크지 못할 것이라고 판단하였고, 당시 팀원들과 자료 수집 및 논의를 통해 확실한 **시스템**으로서의 설계안을 만들기 위해 고민하였습니다.

쏘카는 사내 디자인 툴로 Figma를 활용하고 있었고, 이를 통한 디자인을 개발자가 한땀한땀 구현으로 옮기는 것이 기존 업무 방식이었습니다.

따라서 이 업무 프로세스에 날개를 달아줄 방법들을 위한 고민을 많이 하던 중 몇몇 부분의 최적화가 가능할 것 같다는 판단을 했습니다.

`디자이너 -> 개발자` 로 전달되는 프로세스와 `개발자 -> 개발자` 로 전달되는 프로세스입니다.

이 과정은 Figma의 지원기능을 백분활용하였습니다.

## 2.2 Figma Plugin

`디자이너 -> 개발자`방향으로 소통하는 과정의 최적화 중 일부는 static한 에셋의 처리였습니다.

생각한 구상은 아래 그림과 같았고 이를 위해 Figma plugin을 활용했습니다.

![socar-frame2.0](/img/2026-01-23-socar-frame2-web/socar-frame2.png)

기존의 아이콘 또는 spaicing과 같은 토큰을 업로드 하여 라이브러리에 추가하는 것은 사내 메신저인 슬랙을 통해 추가되었다는 노티와 함께 담당 개발자가 이를 복사하여 관련 레포에 PR을 남기고 검증을 거쳐 배포시점에 맞춰 추가되는 것이 었습니다.

이 과정을 일부 줄일 수 있는 방법으로 내부 플러그인을 배포하여 PAT토큰으로 간단한 검증을 거친뒤 PR생성을 할 수 있도록 하는 내부 플러그인을 개발하여 배포하였습니다.

실제 디자이너는 이 플러그인을 통해 원하는 아이콘을 PR로 바로 전송할 수 있으므로, 전송 후 담당 개발자는 검증만 하면 되는 프로세스로 간소화 할 수 있습니다.

다만 PAT형태의 인증이 사내 Enterprise에 종속되는 사내 플러그인이라 할지라도 보안상으로 적합할 지에 대한 추가 고민이 필요한 상태이며, 이를 위해 추가 플러그인 개발이 필요하다면 별도 인증서버를 두는 방향도 고려하고 있습니다.

![socar-frame2.0](/img/2026-01-23-socar-frame2-web/figma-plugin.png)

이 외에도 토큰과 같은 foundation레벨의 요소가 추가될때는 위와같은 프로세스를 확장시켜 활용할 수 있을거란 생각을 하고 있습니다.

## 2.3 Figma Code Connect

`개발자 -> 개발자`방향으로 소통하는 과정의 최적화 중 일부는 UI 라이브러리 코드 사용법입니다.

기존 디자인시스템의 활용은 스토리북을 통해 UI를 확인하고, 라이브러리를 포함한 레포지토리 내에서 코드를 확인하여 실제 서비스에 구현하는 방식이었습니다.

![figma-code-connect-code](/img/2026-01-23-socar-frame2-web/figma-code-connect-code.png)

하지만 Figma Code Connect를 잘 활용한다면 이 과정을 간소화할 수 있게 됩니다.

![figma-code-connect](/img/2026-01-23-socar-frame2-web/figma-code-connect.png)

사진과 같이 특정 UI를 클릭하면 그에 맞는 UI라이브러리 코드가 나타나게 됩니다.

물론 DatePicker와 같은 복잡한 개념이 산재해 있는 UI의 경우 만족할만한 정합성을 가지기는 쉽지 않았습니다.

그리하여 디자이너와 협업하여 Slot과 상태 활용에 대한 개념을 지속 조율하여 Code Connect시 합성 컴포넌트 형태와 매칭이 가능한 유연하게 되도록 진행하는 과정을 거쳤으며, 이 과정은 상호 다른 직군의 이해도를 바탕으로 이뤄져야 하여 러닝커브가 있었습니다.

# 3. 컴포넌트 설계

## 3.1 설계 목적

컴포넌트 설계의 가장 큰 목적은 재사용성과 독립성으로 잡았습니다. 모든 프론트엔드 개발자들이 겪는 고민인 UI의 변화에 취약한 내부 로직에 있어 각 독립성을 최대한 확보하기 위한 방법을 고려하였습니다.

그 과정에서 많은 오픈소스 라이브러리들의 설계 방식을 차용 및 참고하려 했습니다.

## 3.2 Hook과 객체

구체적인 방향은 UI와 강결합되는 상태를 가지는 컴포넌트와 그렇지 않을 수 있는 컴포넌트를 분류하였습니다.

<!-- 아래는 이유 수정 더 필요함. -->

이렇게 진행한 이유는 기존 컴포넌트들의 재활용이 힘든 선례가 크게 있어 UI/UX 정책을 비즈니스로직을 잘 분리하여 만들어 놓지 않은 경험이 있기 때문입니다.

강결합되는 컴포넌트로 판단되는 경우는 Hook을 통해 리액트 라이프 사이클내의 상태를 활용하는 방식으로 진행하였고,
그렇지 않은 경우는 별도의 객체로 분리하여 Hook을 통한 React Adapter를 두고 UI를 분리하였습니다.

UI와 결합되는 상태를 가지는 컴포넌트는 BottomSheet, Accordion과 같이 UI의 실제 움직임과 상태가 강결합되는 경우 입니다.
BottomSheet의 경우 설계 단계에서 디자이너, 네이티브 분들과 많은 논의 후 상태(hidden, tip, half, max)를 정의할 수 있었고
이에 따라 즉각적으로 UI의 반영과 half 상태에서는 스크롤이 불가한 UX의 일부제한등이 포함되었습니다.

![bottomsheet-ui](/img/2026-01-23-socar-frame2-web/bottomsheet-ui.png)

따라서 BottomSheet과 같은 강결합 케이스는 React의 hook 내부에 비즈니스 로직을 두고 상태변화를 적극 활용하는 방식으로 설계하였습니다.

![bottomsheet-hooks](/img/2026-01-23-socar-frame2-web/bottomsheet-hooks.png)

사진과 같이 `useBottomSheet.tsx`에서 다양한 훅들을 ochestration하여 활용하였습니다.

그렇지 않은 컴포넌트는 UI의 변화와는 별개로 자체에서 처리하는 여러가지 상태를 가진 것으로 보았습니다.

DatePicker와 Pattern(Carousel)이 대표적인 예시였습니다.

![datepicker-ui](/img/2026-01-23-socar-frame2-web/datepicker-ui.png)

DatePicker는 유저에게 즉각적으로 클릭을 통해 날짜를 선택하는 용도의 상호작용되는 start, end, start-end등의 UI상태가 있으나
그 이면에는 선택되지 않은 날짜들의 label과 UI의 표현만을 위한 config등이 존재하였기에 이를 모두 UI의 **상태**와 결합할 필요가 없다 판단하였습니다.

![datepicker-architecture](/img/2026-01-23-socar-frame2-web/datepicker-architecture.png)

React는 UI의 Emit만을 위한 용도로 활용이 가능해졌고 테스트는 용이해지는 장점을 가지게 되었습니다.

단점으로는 러닝커브의 상승과 유지보수의 복잡도가 높아졌다는 단점이 있습니다.

하지만, 테스트코드의 용이성을 통해 안정성을 확보할 수 있어 이 부분을 상쇄할수 있다고 판단하였고 어떤 UI개발 환경에서도 동일한 객체를 활용한다면 동일한 정책을 활용할 수 있다는 장점이 있게되었습니다.

## 3.3 합성컴포넌트와 선언형 메서드

### 합성형 컴포넌트

많은 UI라이브러리에서 활용하는 JSX 기반의 합성컴포넌트 형태를 통해 최대한 자율성을 가지게 하였으며 data attribute의 적절한 배치를 통해 pseudo class를 통한 커스텀 스타일링을 가능하도록 하였습니다.

해당 라이브러리의 1차개발이 끝나기도 전에 몇몇 서비스에서 선반영을 진행하였기에 중간 중간 패치를 해주는 것보다 자율성을 확보시켜주는 방향이 적합하다는 판단이 있어 data attribute를 통한 스타일링 접근을 적극적으로 활용하도록 권장했습니다.

### 선언형 메서드

선언형 메서드는 컴포넌트에 결합하여 각 상태 또는 단계별로 UI를 활용하는 로직이 있는 Alert에 적용하였습니다.

서비스에 주로 에러나 각 요구사항의 단계별로 발생하는 Alert가 다른 경우 Alert내부 컨텐츠를 상태로 하여 활용하는 경우가 많았고 단계가 많아질수록 코드를 통한 로직파악이 힘들다는 단점을 발견하였습니다.

따라서 아래와 같이 open메서드 이후 button을 통해 전달되는 Promise의 resolving되는 형태에 따라 분기를 칠 수 있는 방향으로 메서드를 제공하였습니다.

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

const handleCancle = async () => {
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

    if (result === "cancel") await handleCancle();

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

이 방식을 통해 단계가 깊어져 콜백의 처리가 아쉬울 수 있으나, 기존의 상태를 계속 변경해주는 번거로움을 UI를 통한 선언 형태로 변경할 수 있다는 장점을 취득했다고 생각합니다.

# 4. 패키지 전략

다른 UI라이브러리들과 유사하게 데드 코드를 제거할 수 있는 환경을 제공하였습니다.

초반 tsup을 통한 module형태가 유지되지 않는 형태로 개발되었었으나, 중도에 rollup으로 번들러 전략을 변경하였고 이 과정에서 perserveModule을 활용한 서비스내에서 트리쉐이킹을 위한 Graph 생성에 유리한 구조로 변경하였습니다.

rollup으로 변경한 또 다른 이유는 SSR에 대한 대응이 있었습니다.

쏘카는 현재 next 13~15 버전들이 혼재되어 서비스에서 활용되고 있었고, 13버전의 기본 CSR 지원 형태와 다르게 14~15로 변경되며 `use client`의 상단부 선언에 대한 니즈가 발생하였습니다.

이 과정을 tsup을 통해 진행하기에는 어렵다 판단하여 rollup의 banner메서드를 활용하는 것이 훨씬 간단한 방법이라 판단하여 config의 약간의 복잡도를 높이는 대신 많은 이점이 있다 판단하여 전환을 하였습니다.

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

따라서 위와 같은 useClientBanner메서드와 preserveModules 옵션을 사용하여 변경을 진행하였고, 서비스 내의 번들 전 후 비교를 통해 개선된 수치를 확인할 수 있었습니다.

```md
- 공통 static/chunk의 용량을 1.45MB → 567.07kb 로 약 61%의 감소효과
- first load js는 373kb → 248kb로 33%의 청크 번들 감소효과
- external의 효과로 pages_app-xxx.js로 생기는 큰 청크가 제거됨.
- preserveModule을 통해 트리쉐이킹 친화적 구조가 생성
- 실제 pages와 연결되어있는 chunk파일 확인 시 필요한 컴포넌트만 node_modules안에 빌드된 것 확인
```

![tree-shake](/img/2026-01-23-socar-frame2-web/tree-shake.png)

# 5.AI

## 5.1 Instructions

AI 활용을 위해서는 기본적으로 개발된 UI라이브러리와 Figma를 통해 디자인과 연결된 UI라이브러리의 활용법, 그리고 미쳐 두 파트에서 커버하지 못하는 범위를 instruction으로 넘겨주는 작업을 진행해야 했습니다.

https://socarframe.socar.me 의 문서페이지를 통해서 static한 문서를 제공하는 것을 통해 llm에게 가이드를 주고, 각 서비스의 AGENTS.md에서 이를 활용할 수 있도록 기본적인 가이드를 제공하여 생산성을 증대하는 것을 목표로 했습니다.

![agent-template](/img/2026-01-23-socar-frame2-web/agent-template.png)

실제 배포되어있는 AGENTS.md는 라이브러리 활용을 위한 가이드와 원칙등을 기재하여 LLM이 이를 십분 활용하여 개발을 진행할 수 있도록 했습니다.

![agent](/img/2026-01-23-socar-frame2-web/agent.png)

LLM을 활용한 생산선 개선 측면은 시스템을 설계할때 중요한 하나의 축이었습니다.

글의 초기에 이야기한 `디자인->개발자(라이브러리)->개발자(서비스)`의 흐름에서 사전에 정의된 정책과 코드들을 통해 중간 커뮤니케이션 비용을 확실히 줄일 수 있는 시스템으로서의 역할에 가장 적합하다 판단하였기 때문입니다.

기존의 파편화된 서비스 개발 프로세스의 경우 디자이너가 디자인을 진행, 개발자는 디자이너의 의도를 알기위해 커뮤니케이션 비용을 사용 및 PM은 해당 UI 정책이 기획된 것과 동일한지 판단하는 비용이 낭비되어 왔습니다.

시스템이 자리잡는다면 개발자가 디자이너의 의도를 파악하는것과 일관된 UI의 제공 및 PM은 사전 정의된 UI정책을 활용하면 되므로 커뮤니케이션 비용을 상당히 절약할 수 있게 됩니다.

따라서, UI라이브러리를 넘어 시스템으로 불리기 위해서는 생산성 향상을 위한 기반이 되어야한다고 생각을 하였고, 이를 위해서는 LLM을 잘 활용할 수 있는 환경을 만들려고 했습니다.

잘 알려진 Instruction형태인 AGENTS.md와 llms.txt 등을 LLM활용을 위해 활용하고자 하였으며, 이 글을 쓰는 현재에도 UI라이브러리와 figma code connect를 기반으로하여 사전에 약속된 형태의 Figma설계가 있다면 기본 코드 생성에 있어 나쁘지 않은 생산성을 보여주고는 있습니다.

## 5.2 Figma MCP with LLM

사내 사용되는 다양한 LLM 서비스들에 Figma MCP를 접목한다면 에디터 내에서 특정 노드 ID를 통해 `디자인->개발자(서비스)`로 접근할 수 있게 됩니다.

이 과정에서 지속적으로 변경 및 개선되어야 할 점은 Instruction과 Figma의 설계 방식이라고 생각했습니다.

동일한 디자인 예제라 할지라도 Instruction에 어떤 형태의 지시사항이 적혀있는지에 따라 다른 결과를 나타내는 것을 확인하였고, Figma의 설계에 디자인시스템 반영 여부와 보이지 않는 노드에 대한 처리등에 따라 결과물이 꽤나 상이하였습니다.

이는, 다양한 사례와 더불어 지속적으로 개선이 필요한 부분입니다.

| Instruction개선 전                                         | Instruction개선 후                                           |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| ![ui-first](/img/2026-01-23-socar-frame2-web/ui-first.png) | ![ui-second](/img/2026-01-23-socar-frame2-web/ui-second.png) |

덧붙여, 개발자의 세세한 리뷰와 비즈니스 로직의 코드 반영을 위한 별도의 정책서를 어떤 방식으로 시스템 기반 개발에 녹여낼지에 대해서는 많은 고려가 필요할 것 같습니다.

# 6. 업무진행

디자인 시스템을 위한 개발은 장기간에 걸쳐 진행되었습니다.

각 파트별로 담당하는 팀 또는 인원이 배정되었고, 여러 우선순위들로 인해 우여곡절이 있었으나 큰 틀에서 정기적 회의를 거쳐 UI/UX 정책에 대한 논의를 거치며 합의과정을 만드는 구조로 진행되었습니다.

우선 UI의 우선순위가 정해졌고, N차 개발에 대한 마일스톤이 나눠졌습니다.

그리고 개발이 완료되지 않은 상태에서도 서비스에 이 컴포넌트를 사용하게 해야한다는 요구사항이 있었기에 중간중간 여러 직군의 피드백을 들으며 디자인 정책이 변경되는 경우도 있었습니다.

그러나 큰 틀에서는 합의된 인원이 정책을 확고히 하고, 그 이후 각 담당 개발자들이 개발을 진행한 뒤, QC를 진행하여 상황에 맞는 배포 전략을 가져가는 형태로 진행되었습니다.

이 과정에서 각 OS의 구현 방식에 대한 차이가 다소 있다는 것을 확인하기도 하였고, 상호간의 구조적으로 피할 수 없는 한계도 있다는 것을 알게 되었습니다.

또한, 비 개발직군인 디자이너와 개발직군간의 code-connect과정에서 조율을 하는 과정은 실제로 디자인시스템 개발을 진행하며 경험한 큰 챌린지 중 하나였던 것 같습니다.

# 7. 후기

각 파트별로 아주 많은 이야기를 나눌 수 있는 주제들입니다.

구체적인 코드 구현이나 실제 현상에 대한 이야기보다는 실제로 지향하는 방향과 현재 상태에 대해 공유하고 싶은 마음으로 작성하게 되어 다소 세부 내용이 부족할 수 있으니 너그러운 마음으로 읽어주시길 바랍니다.

아직 SOCAR-FRAME2.0의 메이저 버전이 완전히 개발되지는 않았습니다.

서비스 내에서 사용하는 상위 버전에 대한 지원을 위한 의존 패키지들을 업그레이드한 지원 버전도 필요하고, 언급된 AI 및 Figma Plugin에 대한 보강도 많이 이뤄져야 합니다.

하지만 1차 UI 개발과 시스템 전체의 활용을 위한 기반 개발은 완료되었기에 SOCAR의 클라이언트 개발은 어떤 방향을 가지고 디자인 시스템이라는 것을 개발하게 되었는지에 대해 공유하기 위해 이 글을 작성하게 되었습니다.

여러 동료들의 도움을 통해 해당 개발단계까지 올 수 있게 된 것 같습니다.

이 글을 빌어 함께 진행했던, 진행하고 있는 동료들에게 감사인사를 전합니다.
