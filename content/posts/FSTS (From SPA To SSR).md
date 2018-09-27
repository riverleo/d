---
title: "FSTS (From SPA to SSR)"
date: 2018-10-25T10:09:39+09:00
draft: false
---

FJTR 작업이 마무리된 뒤 한동안 해외진출과 신기능 개발 그리고 업데이트 등으로
바쁜 일정에 쫓기고 있었습니다. 그러던 사이 기술 부채는 눈덩이처럼 불어나 있었고
내부적으로 조직 개편도 여러차례 이어지면서 잦은 개발자 이동으로 히스토리도
상당히 엉켜 있는 상태였습니다.

그러다보니 당시 여러 제품에서 기술 부채를 해결하기 위한 움직임들이 일어났고
그에 대한 일환으로 리팩토링 작업이 내부적으로 많이 진행되고 있었죠. 하지만 그런
분위기 속에서도 유저웹은 상대적으로 사용자가 많고 장애에 민감한 영역이다보니
리팩토링을 진행할 때 생각을 맞추는 과정이 생각보다 쉽지 않았습니다.

꽤 오랜 기간 논의한 끝에 몇가지 이슈를 해결하는 방안으로 리팩토링에 대한 합의에
다다를 수 있었는데 당시 해결하고자 했던 목표는 다음과 같습니다.

- SEO
- 기술 부채
- 속도 개선
- 회색 영역

위 이슈 가운데 일부는 오랫동안 리서치와 준비중이었고 내부적으로 SPA에서 SSR
형태로 전환하는 작업의 필요성도 꾸준히 주장해오고 있었습니다. 그리고 지속적으로
해외 지사에서 속도 관련된 이슈 제기와 마케팅 팀의 SEO 요청, 그리고 서버 개발자
사이의 회색 영역이 섞이면서 문제 해결이 점점 어렵게 되자 자연스럽게 리팩토링에
대한 논의로 이어질 수 있었던 것 같습니다.

중요한 것은 어떻게 진행해 나갈 것인가인데 유저웹은 이전 FJTR 때와는 달리 5개국
(한국, 일본, 대만, 싱가폴, 홍콩)에서 서비스 중이었고 지속적으로 각국 운영팀에
의해 모니터링 되던 상황이다보니 무엇보다 안정성이 중요했습니다. 그 뿐만 아니라
여러 신규 기능들도 계속해서 개발해야 되는 상황이었기 때문에 리팩토링 진행이
FJTR과 비교해서 체감상 수십배는 어려웠던 것 같아요.

개인적으로는 조직이 거대해지고 개발적으로 큰 규모의 리팩토링이 더이상 한 사람의
능력만으로 진행되기 어려워지는 시점에 어떻게 하면 좋을지 많이 고민했던 시기였던
것 같습니다. (당시 고민들은 나중에 시간이 나면 따로 정리해봐야 겠네요.)

### NextJS를 활용한 스트랭글러 패턴

FSTS 리팩토링에서도 FJTR 때와 마찬가지로 스트랭글러 패턴이 사용되었습니다.
스트랭글러 패턴은 한번에 새로운 개발환경으로 이전하는 것이 아니라 점진적으로
이전을 진행할 수 있기 때문에 레거시와 최신 코드를 병행하면서 운영할 수 있다는
장점이 있습니다. 물론 이 점 때문에 리팩토링에 걸리는 소요기간이 상대적으로 길게
느껴지긴 하죠.

그러나 잘 생각해보면 기능이 계속해서 변경되고 추가되는 서비스를 한번에 바꾸는
것은 굉장히 어려운 일입니다. 뎌욱이 리팩토링이 진행되는 중에 기존 서비스에서
기능이 변경되기라도 하면 리팩토링 중인 코드도 계속해서 변경해줘야 하고 어렵사리
완성했다 하더라도 최신 서비스를 기존과 교체하는 과정에서 동시다발적으로 장애가
발생하는 상황을 겪게 되죠.

가장 위험할 때에는 장애는 계속해서 발생하는데 적절한 롤백 시나리오도 없고
어디서 발생하는지 파악도 어려운 상황입니다. 심장이 쫄깃해지는 느낌과 함께
탈지구하는 정신을 경험할 수 있죠. 개발에서 몽환적인 유체이탈을 느끼고 싶다면
추천하지만 기본적으로 큰 규모의 서비스를 이원화해서 리팩토링 하려는 시도는
굉장히 위험하다는 말씀을 드리고 싶어요.

이번 FSTS 리팩토링 과정에서는 스트랭글러 패턴과 더불어 게이트웨이 라우팅 패턴이
함께 사용되었는데 결과적으로 구현한 형태는 다음과 같습니다.

<figure>
  <img src="/images/fsts_1.png" width="531px" />
  <figcaption>FSTS 스트랭글러 외관 구조와 게이트웨이</figcaption>
</figure>

### ALB를 활용한 게이트웨이 라우팅 패턴

우선은 기존 백엔드 서버가 받고 있는 외부요청을 URL에 따라서 SSR용 NextJS 서버로
분산해줄 수 있는 상위 레벨의 라우터가 필요했습니다.

이번 리팩토링은 AWS 인프라가 많이 필요하다보니 서버 분들과 긴밀하게 협조하면서
진행하였는데 역할을 나누어 프론트엔드 팀에서 NextJS 기반으로 스트랭글러 외관을
만드고 동시에 서버 팀에서 ALB를 기반으로 기존 파이썬 서버가 받아야할 요청과 SSR
서버에서 처리할 요청을 분리하는 작업을 해주셨습니다. (진만님 감사합니다.)

이 분리 작업은 게이트웨이 라우팅 패턴을 보면 이해하기 쉽습니다.

<figure>
  <img src="https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gateway-routing.png" width="400px" />
  <figcaption>게이트웨이 라우팅 패턴</figcaption>
</figure>

게이트웨이 라우팅 패턴을 사용하면 기존 URL 구조를 변경하지 않고 하나의 도메인
안에서 리팩토링을 진행할 수 있다는 장점이 있습니다. 사실 기존 URL을 변경한다는
것 자체가 굉장히 어려운 일이긴 하지만 결과적으로 ALB를 라우터로 사용했던 것은
굉장히 좋은 선택이었던 것 같습니다.

ALB는 기존에 사용했던 ELB와는 다르게 호스트와 경로를 옵션으로 활용하여 정교한
분산이 가능했고 인스턴스 그룹을 만들어 그룹 별로 요청을 나눠 전달할 수
있었습니다. 덕분에 서버 간의 독립성을 완벽하게 보장할 수 있어 ASG(Auto Scaling
Group) 등의 기술들을 사용할 수 있었습니다.

<figure>
  <img src="/images/fsts_2.png" width="600px" />
  <figcaption>ALB 웹 관리도구</figcaption>
</figure>

소소하게는 AWS 콘솔에서 제공하는 라우팅 관리도구가 상당히 유용했습니다. 물론
AWS에서 제공하는 관리도구가 정말 잘 만들었다기 보다는 최소한 Nginx 설정파일에서
라우팅을 관리하던 과거에 비하면 천국 같았죠.

### EFS (Elastic File System)

이번 리팩토링 과정에서 서버 개발자 분이 가장 고생해주셨던 부분인 것 같습니다.
중간에 배포방식이나 여러 기술적인 이슈로 조금 좌충우돌하긴 했지만 도입하고 얼마
효과가 가장 좋았던 기술 중 하나입니다.

간단하게 설명하자면 EFS는 서버 인스턴스 간에 동일한 파일 시스템을 사용할 수
있게 해주는 기술입니다.  기존에는 새로운 인스턴스가 뜰 때마다 안에 있는 소스와
설정들을 업데이트 해줘야 하는 과정이 필요했다면 EFS를 도입한 뒤부터는 자동으로
최신 디스크를 사용할 수 있기 때문에 새로 인스턴스를 생성하는 부담감이 크게
줄었습니다.

하지만 무엇보다 EFS를 가장 유용하게 사용했던 부분은 배포였습니다. 보통
프론트엔드에서는 배포 시에 Webpack과 같은 번들러를 사용해 파일을 빌드하는데 이
과정은 시간도 오래 걸릴 뿐더러 CPU와 메모리를 많이 사용하는 까닭에 프로덕션
환경에서는 함부로 실행하기가 어렵습니다. 프로덕션 서버에서 함부로 빌드했다가
서비스가 급격히 느려지거나 죽는 현상이 발생할 수 있죠.

<figure>
  <img src="/images/fsts_3.png" width="566px" />
  <figcaption>EFS 이용한 배포구조</figcaption>
</figure>

결과적으로 기존에는 다른 서버(로컬 또는 배포 서버 등)에서 빌드하고 생성된
번들을 프로덕션 서버에 일일이 넣어줘야 했다면 EFS 환경에서는 한 서버에서 빌드를
하면 자동적으로 다른 모든 서버에 새로 생성된 번들 파일에 접근할 수 있게
되었습니다.

게다가 빌드하는 과정도 기존에 쉬고 있는 테스트 서버에 EFS를 연결하고 진행하면
됐기 때문에 별도의 배포 서버를 운영할 필요도 없고 빌드 과정이 프로덕션 환경에
영향을 주지도 않아 안정적인 배포가 가능했죠.

다만 EFS도 초기에 몇가지 이슈가 있었는데 혹시 도입을 검토 중이시라면 미리 알고
있으면 좋을 것 같아 적어둡니다. EFS에서 처리량 모드를 버스팅 (Bursting)으로
사용할 경우 크레딧 관리가 생각보다 까다롭습니다. 한번은 배포 중에 서버가
재시작하지 않는 이슈가 있었는데 원인을 확인해보니 크레딧이 모두 소진되면서 IO
대역폭이 급격히 떨어지는 이유 때문이더군요.

EFS는 버스팅 모드에서 평소 초당 100mb의 처리량을 유지하다가 `npm install` 등
많은 양의 IO가 발생하는 명령이 실행되면 크레딧이 빠르게 소진되면서 대역폭이
초당 50kb까지 떨어질 수 있습니다. 시간이 지나면 다시 회복되기는 하지만 사용자가
직접 크레딧을 설정할 수 없고 프로비저닝 모드는 운영에 많은 비용이 들기 때문에
가급적 IO를 일정하게 유지할 수 있는 방식으로 운영이 필요한 것 같습니다.

### NextJS

NextJS를 발견하기 전까지는 기존에 사용중인 react-router를 이용해 자체적으로
SSR을 구현할 생각을 하고 있었는데 혹시나 하는 마음에 SSR 관련 라이브러리를
찾아보니 NextJS가 있더군요. 당시에는 몇가지 제약사항 때문에 프로덕션 환경에서는
쓰기 어려웠지만 굉장히 빠른 속도로 발전하는 모습이 당시에 인상적이었습니다.

개인적으로 NextJS의 장점을 말하자면 다음과 같습니다.

##### 풍부한 예제

typescript, sass, redux 등 외부 라이브러리들을 NextJS와 조합하고 싶을 때 어떻게
사용해야 하는지 많은 예제를 통해 설명해주고 있어 비교적 쉽게 적용이 가능합니다.
주요 라이브러리들에 대해서는 공식 플러그인을 제공하고 있는데 생각보다 확장성이
좋아서 사용하기 편리합니다. 처음에는 일일이 수동으로 설정해서 사용하다가 어느덧
공식 플러그인을 사용하게 되었는데 개인적으로 NextJS가 업데이트될 때 호환성을
같이 유지해줘서 좋더군요.

##### next/head

react-helmet 라이브러리를 써본 경험이 있다면 이해가 쉬울 것 같습니다. 사이트
내의 `<head>` 요소를 관리하는데 사용하며 중복된 메타태그에 대해서 가장 마지막에
선언된 메타태그를 반환하도록 설계되었습니다.

```jsx
import Head from 'next/head';

export default () => (
  <div>
    <Head>
      <title>My page title</title>
      <meta name="viewport" content="initial-scale=1.0" key="viewport" />
    </Head>
    <Head>
      <meta name="viewport" content="initial-scale=1.2" key="viewport" />
    </Head>
    <p>Hello world!</p>
  </div>
);
```

위와 같은 코드가 있다면 결과적으로 `<meta name="viewport"
content="initial-scale=1.2" />`를 렌더링합니다.

##### 내장 HMR (Hot Module Replacer)

HMR은 개발 시 코드가 변경되는 것을 감지해 새로고침 없이 새로운 코드를 불러와
주는 도구입니다. 개발할 때 굉장히 편리한 기능이지만 자체적으로 HMR을 구성하는
것은 생각보다 복잡합니다. NextJS에는 HMR 기능이 기본적으로 내장되어 있습니다.

##### Code Splitting과 Dynamic Import

Code Splitting은 사이트 로딩 시에 필요한 코드만 불러오는 기능으로 잘 사용한다면
번들 용량을 크게 줄여 사이트 속도를 개선할 수 있습니다. NextJS에서는 pages
디렉토리를 기준으로 Code Splitting이 적용되도록 기본 설정되어 있습니다.

최신 자바스크립트 스펙인 Dynamic Import 또한 `next/dynamic` 라이브러리로 사용할
수 있는데 최근 6에서 7로 버전이 올라가면서 SASS와 같이 외부 라이브러리도 같이
사용할 수 있게 되었습니다.

개인적으로 NextJS를 사용하는 이유는 편리한 기능을 많이 제공해주는 것도 있지만
무엇보다 높은 확장성이 가장 큰 매력적인 것 같습니다. SSR을 반드시 NextJS로 할
필요는 없지만 원하는 구성으로 확장해서 충분히 사용할 수 있는데 굳이 사용하지
않을 이유도 없는 것 같아요.

### 기술 부채

기술 부채를 어떻게 해결할 수 있을 것인가에 대한 많은 고민과 노력이 오고 가면서
큰 그림에서 테스트와 문서화 그리고 함수형 프로그래밍을 적용하는 것으로 의견을
모으게 되었습니다. 그동안의 경험을 바탕으로 대화나 미팅으로 히스토리를 공유하는
것이 이제는 한계에 다다랐다는 공감대 속에서 가능했던 합의였습니다.

##### jsdoc을 활용한 문서화

문서화는 결론부터 말하자면 여러 라이브러리를 검토하다가 결국 접근성이 좋고
작성하기 쉬운 도구를 쓰자는 취지에서 jsdoc을 선택하게 되었습니다. 요즘 커뮤니티
내에서 많이 쓰이고 있는 storybook이나 react-styleguidist도 검토해 보았지만
별도의 서버를 띄워서 확인해야 했기 때문에 접근성이 많이 떨어져 보였고 몇가지
개발 상 제한을 만드는 부분도 있었습니다.

먼저 react-styleguidist는 컴포넌트의 `static propTypes`를 읽어들여 자동으로
문서화도 해주고 미리보기도 제공해 줍니다. 굉장히 좋은 발상이라고 생각해서 가장
먼저 도입을 검토했는데 HOC(Higher-Order Components)를 제대로 인식하지 못하는
이슈가 있어 사용할 수 없었습니다. 개인적으로 이 이슈만 없다면 굉장히 좋았을 것
같은데 아쉽네요.

```jsx
import HOC from '../HOC';
import { Component } from 'react';

class Card extends Component {
  ...
}

// 아래 같은 경우 react-styleguidist에서 Card 컴포넌트를 인식하지 못합니다.
export default HOC(Card);
```

두번째로 검토한 storybook은 문서화보다는 UI 컴포넌트를 실제 브라우저 환경에서
테스트 하기 적합한 도구였습니다. 예를 들어 특정 조건을 만족시켜야만 보이는 UI
컴포넌트의 경우 storybook을 사용하면 부분적으로 개발 및 테스트하기 좋았습니다.
하지만 개발 시에 그런 조건이 생각보다 많지 않아 활용도가 낮았고 현재 테스트
과정 자체를 전반적으로 QA팀에서 진행하고 있었기 때문에 storybook 도입은 QA팀의
니즈가 있어야만 했습니다.

QA팀은 부분 테스트보다는 최대한 실제와 유사한 환경에서 테스트하는 방식을 더
선호했습니다. 부분 테스트를 하더라도 결국은 전체 플로우도 테스트 해야 하기
때문에 중복되는 태스크에 큰 메리트가 있지는 않았던 거죠. 그렇다고 유저웹
안에서만 사용하기도 애매해서 도입을 보류하게 되었습니다.

최종적으로 jsdoc을 사용하게 되었는데 주석 형태로 사용하기 때문에 특별한 설정
없이도 바로 적용할 수 있었고 다른 프론트엔드 개발자들이 사용하는 개발도구에서
마우스 오버할 때 바로 내용을 확인할 수 있어 접근성이 매우 좋았죠.

<figure>
  <img src="/images/fsts_9.png" width="611px" />
  <figcaption>vscode에서 jsdoc을 확인 하는 방식</figcaption>
</figure>

가장 중요한 것은 생산성이기 때문에 문서화도 모든 파일에 의무화하기 보다는 여러
사람들이 함께 쓰는 코어 모듈이나 코드 리뷰 과정에서 문서화가 필요하다고 의견이
나오는 함수와 컴포넌트에 한해서 진행하고 있습니다. 그 외에는 가급적 개발자의
자율적 판단에 맡기면서 운영하고 있습니다.

##### Jest

예전에 mocha와 karma, istanbul 조합으로 테스트 해보려다가 아무리 해도 동작하지
않아 결국 포기했던 적이 있습니다. 원인을 확인해 보니 모듈 별로 특정 버전에서
호환이 안되는 이슈였는데 그 때부터 여러 테스트 모듈들을 조합해서 사용하는게
조금 꺼려지더군요.

물론 최근에는 많이 개선됐을 수도 있겠지만 그보다는 Jest 하나만 사용하면 다른
그룹에서 만든 모듈들을 조립할 필요없이 All-in-One으로 해결할 수 있다는 점과
특별한 설정 없이 바로 사용 가능하고 watch 모드에서 제공하는 여러 옵션, 그리고
쉽게 커버리지도 리포트를 생성할 수 있다는 점이 마음에 들었습니다.

또 프로젝트 별로 테스트 환경을 나눌 수도 있다는 점도 매우 유용했는데 NextJS
기반으로 코드를 작성하다 보면 때로는 특정 메서드가 서버와 클라이언트에서 동시에
돌아가는 경우가 생깁니다. 이 때 각각의 환경에서 동작하는지 테스트를 구성하는게
생각보다 쉽지 않죠.

```js
// jest.config.js

module.exports = {
  projects: [
    {
      displayName: 'src',
      testEnvironment: 'jsdom',
      testURL: 'http://localhost/',
      testMatch: [...],
      testPathIgnorePatterns: [...],
    },
    {
      displayName: 'server',
      testEnvironment: 'node',
      testMatch: [...],
      testPathIgnorePatterns: [...],
    },
  ],
};
```

Jest에서는 위와 같이 그룹을 나누어 프로젝트를 설정해주기만 하면 테스트할 때
알아서 동일한 함수를 각각 노드와 브라우저 환경에서 테스트 해주더군요.
결과적으로 다음과 같이 서버와 클라이언트 사이드 양쪽에서 동작하는 코드도 쉽게
테스트할 수 있었습니다.

```js
//store.js

const create = (initialState) => {...};

export default (initialState) => {
  if (!process.browser) {
    return create(initialState);
  }

  if (_.isNil(reduxStore)) {
    reduxStore = create(initialState);
  }

  return reduxStore;
};
```

참고로 jsdom을 사용하면 브라우저 쿠키나 `window` 객체도 Mock 형태로 제공해주기
때문에 대부분의 브라우저 환경을 테스트할 수 있습니다.

##### 함수형 프로그래밍과 테스트

함수형 프로그래밍은 lambda에서 영향을 받아 기본적으로 스스로 상태를 가지지 않는
stateless 함수를 뜻합니다. 함수형 프로그래밍을 사용한 대표적인 라이브러리에는
ImmutableJS, lodash 등이 있죠.

올해 초부터 유저웹에서 함께 일하는 개발자 분과 함수형 프로그래밍을 화두로
React 컴포넌트 안에서 적용 가능한 방법론에 대해 자주 논의했었습니다. 함수형
프로그래밍을 사용하는 데에는 저마다 여러 이유들이 있겠지만 개인적으로는 그동안
추상적인 개념으로 받아들여졌던 단일 책임 원칙을 실무에서 적용 가능한 방법론으로
발전시킨 것이라 욕심이 나더군요. 그리고 메서드를 테스트 가능한 단위로 나눠주는
것도 좋았구요.

다만 React 컴포넌트는 스스로 생존주기를 갖고 있고 `this.setState()`와 같이
컴포넌트 자체가 상태값을 사용할 수 있도록 기능을 제공하고 있기 때문에 함수형
프로그래밍으로 React 컴포넌트를 개발하는 것이 적합해보이지 않기도 했습니다.

우선은 간단하게 컴포넌트 생존주기 메서드 안에서 `this.setState()`를 함수형
프로그래밍 방식으로 사용할 수 있는지 확인하기 위해 아래와 같이 인자로 넘겨
보았습니다.

```jsx
import React, { Component } from 'react';
import componentDidMount from './componentDidMount';

class T extends Component {
  componentDidMount = componentDidMount.bind(this, this.setState)

  render() {
    return null;
  }
}

export default T;
```

그러자 다음과 같은 에러가 발생했습니다.

> Warning: Can't call setState (or forceUpdate) on an unmounted component.
> This is a no-op, but it indicates a memory leak in your application. To fix,
> cancel all subscriptions and asynchronous tasks in the componentWillUnmount
> method.

이렇게 경고 문구까지 심어둘 정도면 우리처럼 특이한 방식으로 개발하려는 사람들이
생각보다 많았나 봅니다. 결과적으로 `this.setState()` 함수를 컴포넌트 바깥에서
실행하는 것은 좋지 않은 방식이었고 우회적인 방안이 필요했습니다.

몇번의 논의 끝에 나온 결론은 컴포넌트의 상태값을 redux 저장소로 대체하자는
것이었습니다. 즉 기존의 컴포넌트 상태값을 스토어에 저장해서 사용하는 것인데
redux는 기본적으로 stateless 기반의 모듈이기 때문에 함수형 프로그래밍으로
설계하기 쉬웠고 기존의 `this.setState` 함수들을 최대한 redux 스토어로 운영할
경우 테스트 과정이 단순해지는 이점이 있었습니다.

다만 이 조건에서는 굉장히 많은 스토어들이 생성될 수 있기 때문에 개발자가 제각기
자기만의 방식으로 스토어나 함수들을 만들어 사용할 경우 구조가 굉장히 복잡해질
수 있다는 우려가 있었습니다.

이 과정에서 많은 논의를 거쳐 최종적으로 폴딩 방식의 베스트 프랙티스를 만들어
내었고 아래와 같이 컴포넌트를 폴더 구조로 생성하면서 해당 컴포넌트에서 필요한
함수와 redux를 폴더 구조로 묶는 방식이었습니다.

<figure>
  <img src="/images/fsts_10.png" width="178px" />
  <figcaption>폴딩 방식에서 컴포넌트 구조</figcaption>
</figure>

위의 폴딩 방식의 컴포넌트는 아래와 같이 구현될 수 있습니다.

```jsx
// T/index.js

...

import { connect } from 'react-redux';
import { className } from './index.scss';
import parseHref from './parseHref';
import handleClick from './handleClick';

const mapStateToProps = state => ({
  t: state.t,
});

const T = ({
  t,
  href,
  dispatch,
}) => {
  const { value } = t;
  const parsedHref = parseHref(href);

  return (
    <button
      className={className}
      href={parsedHref}
      onClick={handleClick({
        value,
        dispatch,
        href: parsedHref,
      })}
    >
      ...
    </button>
  );
};

...

export default connect(mapStateToProps)(T);
```

결과적으로 기존 컴포넌트 내에 있던 메서드들을 모두 밖으로 분리하면서 테스트하기
좋은 구조를 만들 수 있었습니다. 게다가 함수 자체에는 아무런 상태값도 담겨있지
않았기 때문에 `dispatch()` 등과 같은 함수는 `jest.fn()`을 이용해 쉽게 확인할 수
있었죠.

```js
import handleClick from './handleClick';

describe('handleClick.js', () => {
  it('dispatch 함수의 호출 여부', () => {
    const dispatch = jest.fn();
    const handle = handleClick({
      href: 'https://www.wanted.co.kr',
      dispatch,
    })();

    // handleClick 함수 내에서 dispatch가 호출되었는지 여부를 테스트한다.
    expect(dispatch).toBeCalled();
  });
});
```

Mock 함수를 사용하지 않고 아래와 같이 Mock 스토어를 연결해서 테스트하는 것도
좋은 방법입니다.

```js
import getStore from '../getStore';
import handleClick from './handleClick';

describe('handleClick.js', () => {
  it('dispatch 함수의 호출 여부', () => {
    const {
      dispatch,
      getState,
    } = getStore();
    const handle = handleClick({
      href: 'https://www.wanted.co.kr',
      dispatch,
    })();

    expect(getState()).toHaveProperty('t.expectedHref', 'expected');
  });
});
```

물론 React를 함수형 프로그래밍 방식으로 개발할 경우 작성해야할 코드도 많아지고
`this.setState()`를 redux 스토어로 대체해서 사용하는 것에서 이질감이 느껴질
수도 있습니다. 그렇기 때문에 유저웹에서도 테스트가 반드시 필요한 코어 모듈
위주로 사용하고 있고 그 외에는 개발자의 자율적인 판단으로 운영하고 있습니다.

하지만 분명한 것은 컴포넌트의 구조를 폴딩 방식으로 변경하고 난 뒤 함수의 역할을
테스트 코드와 jsdoc 문서로 이해하기가 쉬워졌고 이해가 쉬워지면서 자연스럽게
부채로 이어질 수 있는 코드 발생이 적어졌다는 점입니다.

##### Style Pre-processor

최근에 CSS를 그대로 사용하는 경우는 거의 없죠. 많은 분들이 SASS, LESS 문법을
사용하거나 autoprefixer 같은 모듈을 적용하기 위해 CSS 전처리 도구를 사용합니다.
원티드 유저웹에서도 정말 많은 CSS 전처리기를 사용했는데 최종적으로 아래와 같은
구성을 사용하고 있습니다.

- postcss
  - autoprefixer
  - cssnano
- css-modules

수많은 CSS 전처리 도구들을 사용하다 돌고 돌아 결국은 가장 기본으로 돌아온
느낌이네요.

히스토리를 간략히 공유하자면 가장 처음 사용한 라이브러리는 compass였습니다.
많이 고민하면서 적용했던 것은 아니고 기존 레거시 CSS 파일들을 최대한 수정하지
않고 사용할 수 있는 방법을 찾다가 가장 먼저 눈에 띈 라이브러리였죠.

시간이 지나고 sass 파일들이 거대해지면서 compass에서 전처리 속도가 눈에 띄게
느려졌습니다. 가장 느릴 때는 10초 이상 걸렸는데 이 정도 속도에서는 개발 자체가
굉장히 어려웠습니다. sass 파일에서 폰트 크기 하나 바꾸고 확인하는데 10초 이상이
걸렸으니까요.

속도 이슈가 급하다보니 compass에서 node-sass로 교체하기 위해 node-bourbon을
사용했습니다. 덕분에 속도가 눈에 띄게 빨라졌는데 거의 10배 이상 빨라지더군요.
일단 속도 이슈에서 한숨 돌린터라 좀 더 근본적으로 CSS 구조를 개선할 수 있는
라이브러리를 찾았죠.

그 때 사용한 것이 styled-components였습니다. 당시에는 유저웹을 하고 있지 않아서
다른 제품을 개발하면서 사용했는데 CSS를 컴포넌트와 모듈링하기 좋아 차츰 다른
제품에도 사용하면서 결과적으로 프론트엔드 제품 전체에서 styled-components가
메인 전처리기로 사용되었습니다.

하지만 곧 여러 이슈가 생겨나면서 사용하기가 어려워졌는데 styled-components의
가장 큰 문제점은 디버깅이 어렵다는 점입니다. 소스맵을 제공하지 않아 다른 사람이
작성한 sass를 수정하는 일이 굉장히 어려웠습니다.

유저웹은 제품 특성상 다양한 컴포넌트들이 존재하고 sass 수정도 굉장히 잦다보니
CSS를 수정하는 일이 어려울 경우 업무에 큰 지장을 주던 상황이었습니다. 간단한
높이를 수정하는데도 직접 작성한 컴포넌트가 아니라면 해당 라인을 찾고 수정하는데
상당히 오랜 시간이 걸렸죠. 특히 상위 컴포넌트에 작성된 sass가 하위 컴포넌트에도
영향을 주는 경우 단순한 수정에도 컴포넌트의 전체 구조를 파악해야만 진행이
가능했습니다. 최근에 소스맵 지원에 대한 논의가 커뮤니티 내에 진행되고 있는 것
같던데 5.0 로드맵에 포함되어 있더군요. 개인적으로는 소스맵을 지원하기 전까지
사용은 어려워 보입니다.

다른 마이너한 이슈는 styled-components에서 css를 작성할 수 있는  방법이
다양하다보니 개발자마다 구현하는 방식이 달랐고 자바스크립트 코드와 sass 코드가
섞이면서 가독성이 굉장히 떨어지더군요.

가장 치명적이었던 이슈는 styled-components를 중복해서 사용하는 것이 어렵다는
점입니다. 리팩토링 구조에서 스트랭글러 외관을 만들기 위해 두개의 앱을 가진
상태에서 한쪽이 styled-components를 사용하는 경우 반대쪽에서
styled-components를 동시에 사용하려면 기본적으로 불가능하고 서로 의존적인
설정을 통해서만 사용이 가능한데 그 방법이 깔끔하지 않습니다. 내부적으로 v1과
v2 사이에 의존성을 주는 것은 옳지 않다고 판단했고 postcss를 사용하기로
하였습니다.

결과적으로 디버깅이 가장 편하면서 sass를 컴포넌트 단위로 분리해서 사용하기 가장
좋은 라이브러리를 찾다가 postcss를 도입하게 되었습니다. css-modules과 함께
사용하면 js 파일 안에 불러와 파일별로 분리해 사용할 수 있었고 소스맵이 잘되어
있어 개발할 때 수정도 편리하였습니다.

### 캐싱

SSR만 적용해서는 모든 상황에서 속도를 개선할 수 없습니다. 실질적으로 속도를
올릴 수 있는 디테일은 캐싱을 어떻게 사용하는가에 따라 많이 좌우됩니다. 이번에
FSTS 리팩토링을 하면서 어떻게 캐싱을 적용할지 고민하다 결과적으로 in-memory와
redis 저장소를 같이 사용하기로 하였습니다.

redis와 in-memory는 둘다 기본적으로 다른 데이터 저장소에 비하면 빠르지만 가장
빠르고 안정적인 것은 in-memory입니다. 하지만 in-memory는 프로덕션 서버의 자원을
사용하기 때문에 Node 서버를 클러스터 모드로 여러개 생성할 경우 인스턴스 수에
따라 사용량이 급격하게 올라가면서 메모리가 가득 찰 수 있습니다. 기본적으로 모든
데이터를 in-memory에 올려서 사용하는 것은 좋지 않습니다.

그래서 in-memory 저장소는 서비스 전체에 공유되거나 필수적인 데이터만 저장하고
API에서 넘어온 데이터를 그대로 저장하기 보다는 parser를 두어 일정한 캐시 크기를
유지하고 있습니다.

```ts
cache(key: string, cb: () => <T | Promise<T>>, source?: string = "memory": "memory" | "redis") => Promise<T>
```

그외에 각 페이지 또는 특정 컴포넌트에 필요한 데이터는 자유롭게 redis 저장소를
이용하기로 하였습니다. 캐싱 모듈은 저장소 별로 따로 만들지 않고 하나의
인터페이스를 사용하고 있는데 때에 따라 in-memory와 redis를 빠르게 전환하도록
구성하는 것을 염두해 두었습니다.

<figure>
  <img src="/images/fsts_4.png" width="344px" />
  <figcaption>캐시 적용 전</figcaption>
</figure>

NextJS 기반의 스트랭글러 외관을 완성한 뒤에 캐싱이 전혀 적용하지 않고 띄우니
서버로부터 응답시간이 약 3.5초 정도가 걸렸습니다. 로컬 환경에서 측정한 것이기
때문에 프로덕션 환경에서는 이보다 느릴 겁니다. 이렇게 느린 요청은 사용자가 몰릴
경우 병목현상을 발생시키면서 서버를 급격히 느려지게 만들고 CPU 자원을 과도하게
소모하기 때문에 적절한 응답시간을 반환할 때까지 캐싱을 적용해 주어야 합니다.

SSR 서버가 머신러닝이나 빅데이터를 다루지는 않기 때문에 대부분의 병목 현상은
외부 API와 통신하면서 발생합니다. 이런 부분에 적절하게 캐싱을 적용하면 속도를
크게 개선할 수 있습니다.

<figure>
  <img src="/images/fsts_5.png" width="344px" />
  <figcaption>캐시 적용 후</figcaption>
</figure>

속도가 계속 줄어드는 재미에 푹빠져서 한번 될때까지 해보자고 달려드니 25ms까지
줄여지더군요. 이 과정에서는 redis를 쓰지 않고 in-memory 기반으로만 캐싱했기
때문에 프로덕션과 로컬 환경에서 속도 차이가 크게 없었습니다. 덕분에 실제
프로덕션 환경에서도 25~50ms의 속도를 유지하고 있습니다.

<figure>
  <img src="/images/fsts_6.png" width="580px" />
  <figcaption>실제 프로덕션 적용 결과</figcaption>
</figure>

결과적으로 TTFB(Time To First Byte)만 두고 봤을 때 기존과 비교해서 10~20배
정도의 속도 개선되었습니다. 물론 아직 마이그레이션이 진행중인 단계이기 때문에
현재까지만 보자면 페이지 로딩속도를 500ms 정도 단축한 정도입니다. 하지만 모든
페이지가 완전히 v2로 이전했을 경우 첫 화면 자체가 50ms 안으로 뜨는 것까지
기대할 수 있을 겁니다.

### 현재까지 진행 상황

<figure>
  <img src="/images/fsts_7.png" width="588px" />
  <figcaption>올해 3월과 11월 벤치마킹 비교</figcaption>
</figure>

올해 3월 일본을 다녀오며 시작된 속도 개선 작업이 드디어 막바지에 오르고 있는 것
같습니다. 현재까지는 기존과 대비해서 2~3배 정도 개선되긴 했지만 아직은 작업이
많이 남아있는 상태고 더 크게 개선될 여지가 많이 남아있기 때문에 결과를 이야기
하기가 조금 조심스럽네요.

최종적으로 저희가 기대하고 있는 목표는 아래와 같습니다.

<figure>
  <img src="/images/fsts_8.png" width="588px" />
  <figcaption>네이버와 카카오 벤치마킹</figcaption>
</figure>

카카오 페이지 같은 경우 저희와 동일한 NextJS 라이브러리를 사용하여 만들어
졌습니다. 보시면 전체적인 속도에서 네이버가 조금 더 빠른 것처럼 보이지만 첫 로
속도에서 카카오 페이지가 네이버보다 3배 이상 빠르기 때문에 체감할 때에는 카카오
페이지의 속도가 더 빠르게 느껴질 수 있습니다.

저희도 결과적으로 모든 작업이 마무리됐을 경우 오른쪽 카카오 페이지만큼의 속도를
낼 수 있을 것으로 기대하고 있습니다.