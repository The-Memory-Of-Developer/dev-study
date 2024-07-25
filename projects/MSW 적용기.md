# 📓 MSW 적용 배경

## 💡 진행중인 프로젝트 - 스터디 플랫폼

### 🛠 프로젝트 기획의도

- **누구나 쉽고 빠르게 지속 가능한 스터디에 참여할 수 있는 플랫폼 개발**
- **스터디 진행 사항도 관리할 수 있는 플랫폼 개발**

### 🛠 프로젝트 주요 기능

- 스터디 모집
- 스터디 진행 상황 관리
- 스터디 지원

### 🛠 주요 페이지

- **메인 페이지** - 인기있는 스터디 모집 공고를 보여주는 메인페이지
  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/f9e91417-856f-4433-90f9-d93dff9187c1/Untitled.png)
- **모집공고 페이지** - 여러 모집공고를 확인하는 페이지

  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/b77c28d9-334f-40bc-8467-cc10c13c2fec/Untitled.png)

- **모집공고 상세 페이지** - 모집공고의 상세 내용을 확인하는 페이지

  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/41cde457-6247-438a-b32a-dd584514af82/Untitled.png)

- **스터디 상태 페이지** - 진행중인 스터디의 상태를 관리하는 페이지
  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/13061197-0338-4c04-b162-01c5ac5b6719/Untitled.png)
- **마이페이지** - 나의 스터디 정보를 보여주는 페이지
  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/e3f7b665-b68d-45b3-a08e-3d1db4ab3d8a/Untitled.png)

## 💡 프로젝트 진행 시 발생한 문제

### 🛠 맡은 역할

- 메인페이지, 스터디 모집 공고 모아보기 페이지, 스터디 상태 페이지 등
  ⇒ **데이터를 가져와서 처리하여, 화면에 보여주는 기능이 대부분인 페이지**

### 👿 발생한 문제

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/70c7c152-1df6-4ede-a14b-5cef8a375c33/Untitled.png)

- 백엔드에서 API가 나올 때까지 기다려야 한다.
  ⇒ **백엔드 개발에 의존적**
- 개발중에 추가 수정사항이 발생하면, 이러한 의존적인 프로세스를 반복하게 된다.
  ⇒ **FE의 기능 개발 시간 및 테스트 시간 부족**

### 🤔 해결 방안

- 직접 MockData 설정한 상태로 개발 진행
  ⇒ **이는 구현이 쉽다는 장점이 있지만, HTTP 메서드 및 응답 상태에 따른 대응이 쉽지 않다는 단점**
- Mocking 서버를 구현하여 개발
  ⇒ **구현하는데 시간이 소모되며, 실제 서버와 다른 방식으로 동작하여 기존 코드를 수정해야하는 상황 발생**
- **MSW**
  - 서비스 로직을 직접 수정할 필요 X
  - 직접 모킹 서버를 구현할 필요 X
  - 어플리케이션 레벨이 아닌 네트워크 레벨에서 요청을 가로채 응답을 보내기 때문에 모든 종류의 네트워크 라이브러리 (axios, react-query 등) 및 네이티브 fetch 메서드와 사용 가능

## ⇒ MSW 사용

# 📓 MSW 원리

## 💡 MSW(Mock Service Worker)란

- Service Worker를 이용해 서버를 향한 실제 네트워크 요청을 가로채서(intercept) 모의 응답을 보내주는 API Mocking 라이브러리

### 🛠 Service Worker란

- Service Worker는 브라우저가 백그라운드에서 실행하는 스크립트
- 웹 서비스와 브라우저 및 네트워크 사이에서 프록시 서버역할
- 웹 애플리케이션의 메인 스레드와 분리된 별도의 백그라운드 스레드에서 실행되어 애플리케이션의 UI 블록 없이 연산을 처리 가능
  🔎 [Service Worker](https://developer.mozilla.org/ko/docs/Web/API/Service_Worker_API)

## 💡 MSW 작동 원리

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/33a9742e-f4e1-467d-a16e-04175afecc29/Untitled.png)

1. 브라우저가 Service Worker에 Request
2. Service Worker가 해당 요청을 가로채서 복사
3. 서버에 요청을 보내지 않고, 사용자가 작성한 MSW 라이브러리의 handler와 매칭
4. MSW가 등록된 handler에서 Mocked response(모의 응답)를 Service Worker에게 전달
5. Service Worker가 Mocked Response를 브라우저에게 전달

# 📓 MSW 적용

## 💡 MSW 설정

- 메인페이지에서 인기있는 모집공고를 가져오는 API Mocking 작성

### 📋 MSW 관련 폴더 및 파일 구조

```bash
-- public
    -- mockServiceWorker.js => **Service Worker 파일**
		-- index.html
-- src
  	-- mocks
			  -- data => **Mock Data**
			  -- handlers => **Mock Response를 반환하는 handler**
			  -- uitls => **Mock Data를 변환하는 uitil 함수**
			  -- browser.ts => **worker Instance 생성 파일**
    -- App.tsx => **Root Component**
    -- index.tsx => **Entry point**
```

### 🛠 MSW 라이브러리 설치

```bash
yarn add msv --dev
```

### 🛠 Service Worker 등록을 위한 Worker Script 생성

```bash
npx msw init public(지정경로) --save
```

### 🛠 Worker Setup

```tsx
// ../src/Mocks/browser.ts
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);
```

- Mock Response를 반환하는 handlers를 인수로 전달하여 worker Instance 생성

### 🛠 Entry Point에서 Worker 설정

```tsx
// src/index.tsx
import ReactDOM from 'react-dom/client';
import { StrictMode } from 'react';

import App from './App.tsx';

const enableMocking = async () => {
  if (import.meta.env.DEV === false) return;
  const { worker } = await import('./mocks/browser');
  return worker.start();
};

enableMocking().then(() =>
  ReactDOM.createRoot(document.getElementById('root')!).render(
    <StrictMode>
      <App />
    </StrictMode>
  )
);
```

- dev 환경시 worker가 실행될 수 있도록, 애플리케이션 최상단에서 조건부 worker 실행 코드를 설정한다.

### 🛠 인기있는 모집공고 Mock Data 작성

```tsx
//src/Mocks/data/mockData.ts
export const recruitmentDetailMockData: RecruitmentDetailRawDataType[] = [
  {
    id: 1,
    title: '인기 코테 스터디 1',
    stacks: ['Java'],
    positions: ['백엔드'],
    ownerNickname: '휴',
    way: '오프라인',
    startDateTime: '2024-03-01',
    endDateTime: '2024-04-01',
    recruitmentEndDateTime: '2024-02-01',
    createdDateTime: '2024-01-29',
    hits: 111,
    applicantCount: 5,
    platformUrl: 'https://open.kakao.com/o/222222',
    content: 'coding test',
    category: '코딩 테스트',
    platform: '디스코드',
    memberCnt: 4,
    contact: '디스코드',
  },
  {
    id: 2,
    title: '인기 코테 스터디 2',
    stacks: ['Javascript'],
    positions: ['프론트엔드'],
    ownerNickname: 'Hyun',
    way: '온라인',
    startDateTime: '2024-02-03',
    endDateTime: '2024-02-29',
    recruitmentEndDateTime: '2024-02-01',
    createdDateTime: '2024-01-14',
    hits: 222,
    applicantCount: 5,
    platformUrl: 'https://open.kakao.com/o/222222',
    content: 'coding test',
    category: '코딩 테스트',
    platform: '디스코드',
    memberCnt: 4,
    contact: '디스코드',
  },
....
]

export const popularRecruitmentsMockData: PopularRecruitmentsRawDataType = {
  popularCodingRecruitments: [
    ...recruitmentDetailMockData
      .filter((recruitmentDetail: RecruitmentDetailRawDataType) => recruitmentDetail.category === '코딩 테스트')
      .map((recruitmentDetail: RecruitmentDetailRawDataType) => {
			(데이터 변환코드)...
			})
      .sort((a, b) => b.hits - a.hits)
      .slice(0, 3),
  ],
  popularInterviewRecruitments: [
		....
  ],
  popularProjectRecruitments: [
	  ....
  ],
};
```

### 🛠 인기있는 모집공고를 반환하는 handler 작성

```tsx
//src/Mocks/handlers/recruitments.ts
import { HttpResponse, http } from 'msw';
import { popularRecruitmentsMockData } from '../data/mockData';

const baseURL = import.meta.env.VITE_MOCK_API_URL;

const getPopularRecruitments = http.get(`${baseURL}`, () => {
  return new HttpResponse(
    JSON.stringify({ data: popularRecruitmentsMockData, message: 'Success' }),
    {
      status: 200,
      statusText: 'OK',
    }
  );
});
```

### 🛠 인기있는 모집공고를 요청하는 api fetcher 작성

```tsx
// src/apis/recruitments.ts
import { apiRequester } from '@/utils/axios';
import { useInfiniteQuery, useQuery } from '@tanstack/react-query';

export const getPopularRecruitments = () => apiRequester.get('/').then((res) => res.data);
export const usePopularRecruitments = () => {
  return useQuery({
    queryKey: ['popularRecruitments'],
    queryFn: () => getPopularRecruitments(),
  });
};
```

### 🛠 메인페이지에서 api fetcher를 통해 인기 있는 모집공고 Data 가져오기

```tsx
//src/pages/main.tsx
import styled from 'styled-components';
import { bannerDummy } from '../../Shared/dummy';
import Banner from '../../Components/Banner';
import { usePopularRecruitments } from '@/Apis/recruitment';
import { convertPopularRecruitmentsToStudyCardProps } from '@/Utils/propertyConverter';
import CardListInfo from '@/Components/CardListInfo';
import PopularStudyCardList from '@/Components/PopularCardList';
import Button from '@/Components/Common/Button';
import { Up } from '@/Assets';
import UtiltiyButtons from '@/Components/UtilityButtons';

const Main = () => {
  const { data, isLoading } = usePopularRecruitments();
...

  return isLoading ? (
    <div>Loading...</div>
  ) : (
    <MainWrapper>
      <Banner {...bannerDummy} />
      <StudyListWrapper>
        <CardListInfo studyCategory="코딩 테스트" />
        <PopularStudyCardList studyCardsProps={popularRecruitments?.popularCodingRecruitments} />
      </StudyListWrapper>
      <StudyListWrapper>
        <CardListInfo studyCategory="모의 면접" />
        <PopularStudyCardList studyCardsProps={popularRecruitments?.popularInterviewRecruitments} />
      </StudyListWrapper>
      <StudyListWrapper>
        <CardListInfo studyCategory="프로젝트" />
        <PopularStudyCardList studyCardsProps={popularRecruitments?.popularProjectRecruitments} />
      </StudyListWrapper>
      <UtiltiyButtons>
        <Button onClick={handleScroll} className="scroll__btn">
          <Up />
          <span>위로가기</span>
        </Button>
      </UtiltiyButtons>
    </MainWrapper>
  );
};
...
```

## 💡 MSW 적용 결과

### 🛠 Service Worker 실행 여부 확인

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/c9720124-b71e-432b-9a79-6acdcf581679/Untitled.png)

- 개발자 도구의 Application 탭에서 확인한 결과, Service Worker가 잘 작동되고 있음을 확인

### 🛠  Mock Data의 전달 확인

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/ff329ea3-2b93-4e4b-a662-0eca56c899b5/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/70767210-f391-44d2-ae7f-0651ec2b5223/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/6c232503-2a95-4e7f-ae52-6660d38ba4a7/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/88fe22bd-1360-4dde-a1cd-04e635271409/Untitled.png)

- 콘솔을 통해 MSW가 잘 작동되었음을 확인할 수 있다.
- Worker로부터 Mock Response를 통해 인기 있는 모집공고 데이터를 가져온 것을 확인할 수 있다.

### 🛠  메인페이지 렌더링 결과

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/59cd531f-198c-4cff-83ed-a68cf85bd5a4/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/36294cea-5191-4362-9be7-70144e5b4743/46108c93-39e8-4208-b71d-f6ade06b46a9/Untitled.png)

- 받아온 데이터를 화면에 잘 보여주고 있는 것을 확인할 수 있다.

## 💡 MSW 적용 소감

- 공식문서가 잘 정리되어 있어, 빠르게 적용이 가능
- MSW를 통해 별도의 환경 없이 네트워크 요청에 대한 모의 가능
- API에 대한 종속성없이 FE 개발을 진행할 수 있었던 좋은 경험

# 📓 참고자료

- [Mocking으로 생산성까지 챙기는 FE 개발](https://tech.kakao.com/2021/09/29/mocking-fe/)
- [Service Worker](https://developer.mozilla.org/ko/docs/Web/API/Service_Worker_API)
- [MSW(Mock Service Worker)로 더욱 생산적인 FE 개발하기](https://velog.io/@khy226/msw%EB%A1%9C-%EB%AA%A8%EC%9D%98-%EC%84%9C%EB%B2%84-%EB%A7%8C%EB%93%A4%EA%B8%B0#service-worker%EB%9E%80)
- [MSW](https://mswjs.io/)
