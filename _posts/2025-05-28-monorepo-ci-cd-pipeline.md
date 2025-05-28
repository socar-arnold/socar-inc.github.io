---
layout: post
title: "turborepo 및 캐시서버와 함게하는 파이프라인 개선하기"
subtitle: FE Core팀의 파이프라인 개선기
date: 2025-05-28 00:00:00 +0900
category: data
background: "/img/fe-turborepo-pipeline/matrix.png"
author: arnold
comments: true
tags:
  - frontend
  - github workflow
  - turborepo
  - monorepository
---

1. [들어가며](#1-들어가며)
2. [기존 파이프라인의 필요성과 문제점](#2-기존-파이프라인의-필요성과-문제점)
   - [2.1 mono-repository 구조에서 다양한 프로젝트 운영 시 필요한 파이프라인](#21-mono-repository-구조에서-다양한-프로젝트-운영-시-필요한-파이프라인)
   - [2.2 파이프라인 진행 시간에 대한 문제점](#22-파이프라인-진행-시간에-대한-문제점)
3. [파이프라인 개선을 위한 아이디에이션](#3-파이프라인-개선을-위한-아이디에이션)
   - [3.1 파이프라인을 구성할 재료 모으기](#31-파이프라인을-구성할-재료-모으기)
   - [3.2 재료 조합 샘플 만들어 보기](#32-재료-조합-샘플-만들어-보기)
4. [최종본 개발 및 운영 적용](#4-최종본-개발-및-운영-적용)
   - [4.1 matrix와 dryrun 그리고 remote cache서버 적용](#41-matrix와-dryrun-그리고-remote-cache서버-적용)
5. [결과 및 후기](#5-결과-및-후기)

# 1. 들어가며

안녕하세요 FE Core팀 아놀드입니다.

쏘카 프론트엔드는 다양한 구조를 가지고 있으나, 최근 가장 배포가 잦게 일어나고 변동사항이 많은 레포지토리 하나가 있습니다.

해당 레포지토리는 webview 내에 존재하는 페이지들을 관리하기 위해 사용되며 turborepo를 활용한 mono-repository 구조 및 pnpm을 통한 패키지 관리를 하고 있습니다.

상용 프로젝트들이 즐비한 상태에서 default branch인 main으로의 머지가 다른 프로젝트에 영향을 끼치지 못하도록 신경을 써야만 하는 구조가 되었고, 사람이 직접 신경쓰는 것을 줄이고 자동으로 휴먼 에러를 잡기 위해 다양한 workflow가 존재하게 되었습니다.

이번 글에서는 그 중 가장 시간을 많이 잡아먹고, 중요하다 판단되는 CI 파이프라인 중 하나를 개선한 경험을 공유하겠습니다.

## 2.기존 파이프라인의 필요성과 문제점

### 2.1 mono repository 구조에서 다양한 프로젝트 운영 시 필요한 파이프라인

현재 turborepo를 이용한 mono repository에는 30여개의 상용 프로젝트가 존재합니다.

각 프로젝트는 담당 팀에 따라 독립적으로 운영중이고, 별도의 배포일정을 가지게 되므로 main으로의 머지는 굉장히 잦고, 그로 인해 `pnpm-lock.yaml` 또한 굉장히 자주 바뀌게 되었습니다.

`pnpm-lock.yaml`의 변화를 방지하고자 caret의 사용을 자제하거나 core 라이브러리를 정해 일관성을 유지하려는 노력도 기울였지만 하위 의존되는 패키지들의 caret으로 인한 변화는 막을 수 없기에 `pnpm-lock.yaml`의 변화는 불가피하다 판단하였습니다.

따라서 main으로 머지되는 경우에 `pnpm-lock.yaml`이 변경된다면 의존성을 가질 수 밖에 없는 모든 프로젝트들의 빌드를 실행하고 안정성에 대한 체크를 해야만 하는 구조가 되었습니다.

### 2.2 파이프라인 진행 시간에 대한 문제점

기존의 파이프라인은 turborepo를 십분 활용하기 위해 **원격 캐시서버**를 k8s환경에 별도로 구성 및 유지 관리 하고 있었고 **`dorny/paths-filter@v3` 를 활용해 불필요한코드 변경점에 대한 빌드를 지양하도록** 설정 해놓은 상태 였습니다.

그럼에도 캐시가 없는 경우 30여 개의 next.js앱을 빌드하는 시간은 **20분 이상**이 걸렸으며, 동시에 여러가지 워크플로우가 실행되는 경우 실제 브랜치 병합까지는 최장 30분이상에 달하는 **생각보다 오랜시간**을 대기해야했습니다.

## 3. 파이프라인 개선을 위한 개발

개선에 앞서 어떤 부분을 어떻게 개선해야 `이 워크플로우가 안전하고 빠르게 작동될까?` 에 대해 고민을 했습니다.

상용 프로젝트의 배포 안정성을 위해 필수적으로 사용하고 있지만 너무나 긴 실행시간을 가졌기 때문입니다.

이 고민에서 빌드 단계를 무조건 거쳐야만 한다면, 빌드 시간을 해결해야 한다고 판단했고 이를 해결할 수 있는 지점은 크게 두가지로 나타났습니다.

1. **ubuntu 러너의 적은 메모리 개선**
2. **빌드 확인 단계의 오랜 실행 시간 개선**

아래 사진을 보면 `32m 42s` 나 되는 러닝타임을 가집니다.

![image.png](attachment:81499794-6792-4e31-9d71-c4a73afed08d:image.png)

해당 워크플로우는 결국 빌드 확인 단계에서 `Error: The operation was canceled.` 를 내뱉으며 워크플로우가 강제 중단 되어 버렸습니다.

우선 할 수 있는 파이프라인의 성능을 올리는 일부터 진행해보기 위해, 코어를 올린 새로운 Runner를 생성하였습니다.

### 3.1 Runner 변경

- 기존 Runner

![image.png](attachment:b7d440d5-9d5c-497b-b06d-170c319c201b:image.png)

- 변경된 Runner

![image.png](attachment:cafbed03-b181-4b2e-9231-9fce67b4fa3f:image.png)

Runner를 변경하며 파이프라인의 성능을 개선하고 나서 평균 20분 이상 돌아가던 워크플로우의 실행 시간을 **10분**대로 줄일 수 있었습니다.

그 결과로 빌드시간이 오래 걸려도 사진의 예시와 같이 워크플로우가 **중단되어버리는 케이스**는 없앨 수 있게 되었습니다.

### 3.2 빌드 확인단계 변경

이미 10분여 이상 감소한 워크플로우의 시간을 확인할 수 있었지만, 몇가지 개선이 필요한 지점은 추가로 있다고 판단했습니다,

우선, 30여개의 next js 프로젝트라 하더라도 10분은 너무 긴시간이라 판단했습니다.

추가로 어떤 프로젝트의 빌드가 진행되고 어떤 프로젝트가 남이있는지에 대한 가시성이 부족했습니다.

기존 워크플로우에서 모든 프로젝트의 빌드를 확인하는 방법은 아주 간단했습니다.

```jsx
- name: 빌드 확인
    if: steps.changes.outputs.with_build == 'true'
    run: pnpm build
```

`pnpm build` 커맨드를 통해 turbo build를 실행하고 이 때, 캐시서버를 통해 저장된 빌드 캐시 아티팩트를 활용합니다.

따라서, 최대한 가시성을 확보하며 병렬적으로 빌드를 처리하고 불필요한 빌드는 건너뛸 수 있는 아이디어가 필요했습니다.

그리하여, 처음 적용하게 된 개념은 `github workflow matrix`라는 개념입니다.

matrix에 들어간 다중의 요청들을 병렬로 실행시켜주는 개념이며 구체적인 설명은 [공식문서](https://docs.github.com/ko/enterprise-cloud@latest/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow)를 참고하는 것을 강력히 추천드립니다 :)

병렬로 각 프로젝트를 빌드하는 경우, 실패 또는 성공에 대한 판단을 가시적으로 할 수 있어 좋은 수단이라 생각하였고 아마도 시간을 더 줄여줄 수 있지 않을까 하는 기대도 하게 되었습니다.

그 이유는, turborepo의 원격 cache server가 활용되고 있기에 하나의 매트릭스 요소의 빌드만 진행 되어도 프로젝트에서 공통으로 의존하고 있는 패키지들에 대한 캐시가 업로드 될 것이고 나머지 요소들은 이 캐시를 활용할 수 있다고 판단했기 때문입니다.

따라서, 매트릭스 적용으로 인해 다중 빌드를 한다고 캐시를 활용하지 못하는 경우는 없다고 판단했습니다.

그래서 간단한 쉘 스크립트를 작성하여 apps 하위의 모든 프로젝트들을 가져오고 해당 프로젝트 별로 빌드를 진행시키는 방식으로 워크플로우를 변경하였습니다.

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

위와 같이 변경하였을 때, 아래와 같은 결과를 얻을 수 있었습니다.

| 비고                                     | 기존 전체 빌드                                                                                 | matrix적용 후                                                                 | 차이               |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------ |
| 캐시 없는 경우 빌드 시간                 | [10m27s](https://github.com/socar-inc/monorepo-socar/actions/runs/13666548600/job/38211258725) | [8m6s](https://github.com/socar-inc/monorepo-socar/actions/runs/13667641032)  | **-2m21s(-22.5%)** |
| 캐시 있는 경우 빌드 시간(전체 캐시 히트) | [1m14s](https://github.com/socar-inc/monorepo-socar/actions/runs/13667556132)                  | [6m57s](https://github.com/socar-inc/monorepo-socar/actions/runs/13667759972) | **+5m43s(463.5%)** |

캐시가 없는 경우는 다소 유리하나 캐시가 있는 경우에도 **병렬처리를 위한 매트릭스 세팅**에 시간이 들어 개선이라 하기 민망한 변경이 되었습니다.

결국 매트릭스의 생성이 문제라면 캐시가 있는 경우 매트릭스 생성을 멈추면 어떨까라는 아이디어에서 시작된 팀 내의 논의에서 turborepo의 특정 플래그 에 대한 이야기가 나왔습니다.

`turbo run build --dry-run` 플래그를 통해 package 또는 app이 캐시 되었는지, 되어있지 않은지에 대한 여부를 미리 판단해 볼 수 있는 기능입니다.

이 기능을 활용한다면 캐시가 있는 경우, 불필요한 matrix생성을 막을 수 있을 것 같았고 가설은 아래와 같았습니다.

![image.png](attachment:1e0b1d0b-cca3-4d9c-9926-9561bc21f91a:image.png)

아래와 같이 turborepo의 캐시를 dry run을 통해 확인하고, 모두 캐시가 되어있다 판단 되는 경우 다음 단계를 미 진행합니다.

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

몇 몇 시도를 하며 단점을 상쇄하는 방법을 고민하며 생각한 가설과 같이 dry run을 통해 캐시가 되지 않은 프로젝트 또는 패키지만 수집하고 빌드를 matrix를 통해 병렬로 진행하는 구조를 만들게 되었습니다.

시간의 변화는 아래와 같이 개선 되었습니다.

| 비고                                     | 기존 전체 빌드                                                                                 | matrix+dry 적용 후                                                            | 차이               |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------ |
| 캐시 없는 경우 빌드 시간                 | [10m27s](https://github.com/socar-inc/monorepo-socar/actions/runs/13666548600/job/38211258725) | [5m29s](https://github.com/socar-inc/monorepo-socar/actions/runs/13691830271) | **-4m58s(-47.5%)** |
| 캐시 있는 경우 빌드 시간(전체 캐시 히트) | [1m14s](https://github.com/socar-inc/monorepo-socar/actions/runs/13667556132)                  | [1m11s](https://github.com/socar-inc/monorepo-socar/actions/runs/13691965986) | **-3s(-4%)**       |

캐시가 없는 경우의 빌드 시간은 **workflow matrix**를 통해 줄일 수 있는 장점을 살리고, 캐시가 있는 경우 **turborepo의 캐싱 기능**을 그대로 활용하며 워크플로우가 더욱 견고하고 시간을 덜 잡아먹도록 변경을 완료하게 되었습니다.

## 4. 결과 및 후기

하나의 워크플로우 개선을 위해 Runner를 업그레이드하고 workflow의 matrix개념과 turberepo의 캐시서버 및 dry run 플래그 까지 활용하는 등 다양한 요소들을 섞어 원하는 결과를 도출하였습니다.

지금은 단지 10~20여분의 시간을 감소시켰으나, mono repository내의 프로젝트가 많아지면 많아질 수록 이번 개선은 더욱 큰 힘을 발휘할 것 같습니다.

PS. 함께 고민과 개발을 진행한 FE Core팀의 하루, 리우 감사합니다.
