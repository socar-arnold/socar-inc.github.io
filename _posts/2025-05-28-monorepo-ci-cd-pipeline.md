---
layout: post
title: "turborepo와 함게하는 CI 파이프라인 개선하기"
subtitle: FE Core팀의 파이프라인 개선기
date: 2025-05-28 00:00:00 +0900
category: fe
background: "/img/2025-05-28-monorepo-ci-cd-pipeline/matrix.png"
author: arnold
comments: true
tags:
  - github workflow
  - turborepo
  - monorepository
---

<br />

# 목차

1. [**들어가며**](#1-들어가며)
2. [**기존 파이프라인의 필요성과 문제점**](#2-기존-파이프라인의-필요성과-문제점)
   - [**2.1 mono repository 구조에서 다양한 프로젝트 운영 시 필요한 파이프라인**](#21-mono-repository-구조에서-다양한-프로젝트-운영-시-필요한-파이프라인)
   - [**2.2 파이프라인 진행 시간에 대한 문제점**](#22-파이프라인-진행-시간에-대한-문제점)
3. [**파이프라인 개선을 위한 개발**](#3-파이프라인-개선을-위한-개발)
   - [**3.1 Runner 변경**](#31-Runner-변경)
   - [**3.2 빌드 확인 단계 변경**](#32-빌드-확인-단계-변경)
4. [**결과 및 후기**](#4-결과-및-후기)

---

<br /><br />

# 1. 들어가며

안녕하세요 FE Core팀 아놀드입니다.

쏘카 프론트엔드는 다양한 구조를 가지고 있으나, 최근 가장 배포가 잦게 일어나고 변동사항이 많은 레포지토리 하나가 있습니다.

이 레포지토리는 webview 내 여러 페이지를 관리하며, turborepo 기반의 mono-repository 구조와 pnpm 패키지 관리 방식을 적용하고 있습니다.

여러 상용 프로젝트가 함께 존재하는 환경에서는 main 브랜치로의 병합이 다른 프로젝트에 영향을 주지 않도록 신경 써야 합니다. 이를 위해 다양한 workflow를 통해 자동화와 안정성을 확보하고 있습니다.

이번 글에서는, 특히 시간 소요가 크고 중요한 CI 파이프라인을 개선한 경험을 정리해 공유합니다.

## 2.기존 파이프라인의 필요성과 문제점

### 2.1 mono repository 구조에서 다양한 프로젝트 운영 시 필요한 파이프라인

현재 turborepo 기반 mono repository에는 30여 개의 상용 프로젝트가 포함되어 있습니다.

각 프로젝트는 담당 팀에 따라 독립적으로 운영중이고, 배포일정도 다릅니다.
이로 인해 main branch로의 병합이 빈번하게 발생하고, 그로 인해 `pnpm-lock.yaml` 또한 자주 변경 되었습니다.

의존성 변화를 줄이기 위해 caret 사용을 최소화하고 core 라이브러리를 고정하는 등의 시도를 했지만, 하위 패키지의 caret 사용으로 인한 변화까지 막기는 어려웠습니다.
결국 `pnpm-lock.yaml`의 변경은 피할 수 없는 상황이 되었습니다.

따라서 main 브랜치 병합 시, `pnpm-lock.yaml`이 변경되었다면 의존성이 있는 모든 프로젝트에 대해 빌드와 안정성 검증이 필요했습니다.

### 2.2 파이프라인 진행 시간에 대한 문제점

기존의 파이프라인은 turborepo의 활용 및 보안과 확장성을 위해 **원격 캐시서버**를 k8s환경에 별도로 유지 관리 하고 있었습니다.
또한, **`dorny/paths-filter@v3` 를 활용해 불필요한코드 변경점에 대한 빌드를 지양하도록** 설정 해놓은 상태 였습니다.

그럼에도 캐시가 없는 경우 30여 개의 Next.js앱을 빌드하는 시간은 평균 **20분 이상**이 소요되었습니다.
여러가지 워크플로우가 동시에 실행되는 경우 브랜치 병합까지는 최장 **30분 이상**에 달하는 시간을 대기해야했습니다.

## 3. 파이프라인 개선을 위한 개발

개선에 앞서 어떤 부분을 어떻게 개선해야 `이 워크플로우가 안전하고 빠르게 작동될까?` 에 대해 고민을 했습니다.

배포 안정성을 확보하기 위해 빌드 단계를 생략할 수는 없었으므로, 빌드 시간을 단축하는 것이 핵심 과제였습니다.

특히 다음 두 가지를 개선 대상으로 설정했습니다.

1. **Ubuntu Runner의 메모리와 코어 사양 상향**
2. **빌드 확인 단계의 실행 시간 단축**

아래 예시처럼 워크플로우 실행 시간이 `32`분을 넘기는 경우도 있었습니다.

![runtime.png](/img/2025-05-28-monorepo-ci-cd-pipeline/runtime.png)

결국, 빌드 확인 단계에서 `Error: The operation was canceled.` 오류와 함께 강제 종료되는 사례가 반복됐습니다.

이에 따라, 먼저 Runner 성능을 개선하는 작업부터 진행했습니다.

### 3.1 Runner 변경

- 기존 Runner

![originrun.png](/img/2025-05-28-monorepo-ci-cd-pipeline/originrun.png)

- 변경된 Runner

![changerun.png](/img/2025-05-28-monorepo-ci-cd-pipeline/changerun.png)

Runner의 사양을 상향 조정한 결과, 파이프라인 실행 시간이 기존 20분대에서 10분대로 단축되었습니다.

이후로는 빌드 시간이 길어도 **워크플로우가** **중단되어버리는 케이스** 없이 안정적으로 동작하게 되었습니다.

### 3.2 빌드 확인단계 변경

Runner 성능 개선 이후에도 30여 개의 Next.js 프로젝트 빌드에 10분이 소요되었습니다.

여전히 빌드 시간이 길고, 각 프로젝트의 빌드 진행 상황을 파악하기 어렵다는 한계가 있었습니다.

기존 워크플로우에서는 아래와 같이 모든 프로젝트를 한번에 빌드했습니다.

```jsx
- name: 빌드 확인
	  if: steps.changes.outputs.with_build == 'true'
	  run: pnpm build
```

이 방식은 turbo의 원격 캐시 서버를 활용하지만, 빌드 과정을 병렬 처리하거나 가시성을 확보하는 데는 한계가 있었습니다.

따라서 GitHub Workflow [Matrix](https://docs.github.com/ko/enterprise-cloud@latest/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow)를 도입해, 각 프로젝트별로 병렬 빌드를 수행하는 방식으로 개선했습니다.

Matrix 전략을 활용하면, 여러 프로젝트의 빌드를 동시에 실행하면서 각 빌드의 성공 여부를 개별적으로 확인할 수 있습니다.

이때, turborepo의 원격 캐시 서버가 이미 구축되어 있기에 하나의 빌드 결과가 곧바로 다른 프로젝트 빌드에도 활용될 수 있습니다.

따라서, Matrix의 빌드 대상을 생성하는 쉘 스크립트를 별도로 구성하였습니다.

그리고 아래와 같이 apps 하위 프로젝트들을 자동으로 탐색해 각 프로젝트별 빌드를 병렬로 실행하도록 워크플로우를 구성했습니다.

```yaml
---
jobs:
  generate-matrix:
    runs-on: ubuntu-22.04-cores
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - name: 저장소 체크아웃
        uses: actions/checkout@v4
      - name: 빌드 대상 패키지 목록 생성
        id: set-matrix
        run: echo "::set-output name=matrix::$(bash .github/scripts/get-packages.sh)"

  build:
    needs: [generate-matrix]
    runs-on: ubuntu-22.04-cores
    strategy:
      matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}
      fail-fast: false
```

변경된 워크플로우의 결과는 아래와 같습니다.

| 비고                                     | 기존 전체 빌드 | matrix적용 후 | 차이               |
| ---------------------------------------- | -------------- | ------------- | ------------------ |
| 캐시 없는 경우 빌드 시간                 | 10m27s         | 8m6s          | **-2m21s(-22.5%)** |
| 캐시 있는 경우 빌드 시간(전체 캐시 히트) | 1m14s          | 6m57s         | **+5m43s(463.5%)** |

캐시가 없는 경우는 유리하지만 캐시가 있는 경우에도 **병렬처리를 위한 매트릭스 세팅**에 시간이 들어 개선이라 하기에는 아쉬운 변경이 되었습니다.

이 문제를 해결하기 위해, 캐시가 모두 적용된 경우에는 matrix 생성을 건너뛰는 방안을 논의했습니다.

turborepo의 `turbo run build --dry-run` 플래그를 사용하면, 각 패키지나 앱이 캐시되어 있는지 미리 확인할 수 있습니다.

커맨드를 실행시켜보면 프로젝트별로 아래 사진과 같은 정보가 나타나게 됩니다.

![commandinfo.png](/img/2025-05-28-monorepo-ci-cd-pipeline/commandinfo.png)

빌드 해시값과 캐시의 여부를 미리 판단해줍니다.

이 기능을 활용한다면 캐시가 있는 경우, 불필요한 matrix생성을 막을 수 있을 것 같았고 가설은 아래와 같았습니다.

![idea.png](/img/2025-05-28-monorepo-ci-cd-pipeline/idea.png)

모든 프로젝트가 캐시되어 있다면 빌드 단계 자체를 건너뛰고, 그렇지 않은 경우에만 캐시가 되지 않은 프로젝트만 추려서 matrix 빌드를 병렬로 실행하도록 구조를 변경했습니다.

아래 코드와 같이 turborepo의 캐시를 dry run을 통해 확인하고, 모두 캐시가 되어있다 판단 되는 경우 다음 단계를 미 진행합니다.

그렇지 않은 경우, 캐시가 되지 않은 것들만 matrix배열에 담아 병렬적으로 빌드를 진행합니다.

```yaml
#...코드 변경점 확인 및 prebuild 설정 진행
- name: turborepo 캐시 확인
  id: check-cache
  run: ./.github/scripts/check-turborepo-cache.sh
- name: FULL TURBO
  if: ${{ steps.check-cache.outputs.packages-to-build-length == 0 && steps.changes.outputs.with_build != 'true' }}
  run: echo "모든 빌드 대상이 캐시되어 있어, 진행하지 않습니다."

#...기타 환경변수 세팅

- name: ${{ matrix.package }} 프로젝트 빌드
	run: pnpm build --filter=${{ matrix.package }}...
```

이렇게 개선한 결과, 시간의 변화는 아래와 같이 개선 되었습니다.

| 비고                                     | 기존 전체 빌드 | matrix+dry 적용 후 | 차이               |
| ---------------------------------------- | -------------- | ------------------ | ------------------ |
| 캐시 없는 경우 빌드 시간                 | 10m27s         | 5m29s              | **-4m58s(-47.5%)** |
| 캐시 있는 경우 빌드 시간(전체 캐시 히트) | 1m14s          | 1m11s              | **-3s(-4%)**       |

캐시가 없는 경우의 빌드 시간은 **workflow matrix**를 통해 줄일 수 있는 장점을 살리고, 캐시가 있는 경우 **turborepo의 캐싱 기능**을 그대로 활용하였습니다.

그 결과로, 워크플로우가 더욱 견고하고 시간을 덜 잡아먹도록 변경을 완료하게 되었습니다.

### 3.3 매트릭스 완료 단계 추가

Workflow matrix를 활용하면, 각 프로젝트별로 별도의 status check를 위한 CI 옵션들이 생성됩니다.

![matrix.png](/img/2025-05-28-monorepo-ci-cd-pipeline/matrix.png)

브랜치 보호를 위해 단일 status로 체크하려면, 모든 matrix 빌드가 완료된 후 별도의 단계에서 검증하도록 설정해야 합니다.

이를 위해, matrix 빌드 이후에 최종 확인 단계(verify-build)를 추가하여 모든 빌드가 성공적으로 완료되었는지 검증하는 구조로 변경했습니다.

기존의 단계에 추가로 아래의 코드를 추가하여 브랜치 안정성을 추가할 수 있게 되었습니다.

```jsx
  verify-build:
    needs: build-matrix
    runs-on: ubuntu-22.04-4-cores
    name: 'verify-build'
    steps:
      - name: 모든 빌드 성공 확인
        run: echo "모든 build-matrix 작업이 성공적으로 완료되었습니다."
```

## 4. 결과 및 후기

이번 CI 파이프라인 개선에서는 Runner 성능 업그레이드, workflow matrix 적용, turborepo 캐시 서버 및 dry run 기능까지 다양한 기법을 조합해 최적의 결과를 얻을 수 있었습니다.

현재는 10~20분 수준의 빌드 시간을 절감했지만, mono-repository 내 프로젝트가 더 늘어날수록 이번 개선의 효과는 더욱 커질 것입니다.
