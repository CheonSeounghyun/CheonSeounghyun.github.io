---
layout: post
date: 2025-05-29
title: "CursorAI를 사용해서 1800줄이 넘는 JS코드 리팩토링하기"
published: true
lang: ko
excerpt: AI 에이전트 툴을 사용해 소스코드를 유지보수 하기 쉽게 수정해 봅니다
tags: ai
author: seounghyun
---

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-01.png" | absolute_url}}){: .center-image height="370px" }

프로그래밍을 하다 보면 시간이 지날수록 점점 복잡해지고 길어지는 코드에 부담을 느낄 때가 많습니다. 서울 데이터 허브의 챗봇 기능을 구현한 하나의 JavaScript 파일이 1800줄이 넘게 되면서, 점점 수정이나 유지보수가 어려워지고 있다는 걸 느꼈습니다. 그래서 이번에 Cursor AI라는 도구를 활용해 이 긴 코드를 체계적으로 리팩토링해보기로 했습니다. 처음엔 조금 막막했지만, AI의 도움을 받으면서 코드의 구조를 다시 정리하고, 기능별로 나누며, 더 읽기 쉽고 관리하기 쉬운 형태로 바꿔나갈 수 있었습니다. 이 게시글에서는 그 과정에서 겪은 시행착오와 배운 점들, 그리고 Cursor AI를 어떻게 활용했는지를 천천히 이야기해보려 합니다.

## ChatbotLlm.js
chatbotLlm.js 파일은 서울데이터 허브의 챗봇 화면에 대한 모든 코드를 담고 있는 하나의 js 파일입니다. 전역변수, 렌더링 함수, 검증 함수, 유틸 함수, 이벤트등 화면 구성 및 동작에 필요한 모든 코드가 하나의 js 파일에 작성되어 있었습니다. 챗봇 화면을 고도화 하는 개편작업을 위해 코드를 분석 및 수정하면서, 각 기능별로 다른 속성의 함수들이 한 파일에 작성되어 있어 작업함에 불편함이 있었습니다. 참조하는 곳을 찾기 위해 파일을 긴 스크롤을 위아래로 움직이고 적용되는 함수를 찾다보니 매우 헷갈렸습니다. 우선적으로 기존 코드에 수정작업을 진행해 업무 목표는 달성했지만, 앞으로의 유지보수 관리 측면에서 해당 파일을 분리하고 리팩토링하는게 최우선일 것이라 생각했습니다.

## ChatGPT
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-02.png" | absolute_url}}){: .center-image }

처음에는 ChatGPT를 활용해 리팩토링을 시도해보았습니다. 프로젝트에 해당 JavaScript 파일을 추가하고, 전체 코드를 분석해달라고 요청했습니다. ChatGPT는 파일의 전반적인 구조를 빠르게 파악하고 주요 기능들을 구분해 설명해주는 데에 꽤 도움이 되었습니다. 특히 어떤 부분에서 코드가 중복되고 있고, 어떤 함수들이 너무 많은 역할을 하고 있는지에 대해 명확하게 짚어주었습니다. 파일로 다운로드 받을 수 있게 요청했더니 실제로 다운로드 받을 수 있는 링크를 제공했습니다. JS 파일을 분리해 구성했다는 목록을 보니 잘 진행된 것같아 파일을 다운받아 열어 보았습니다만...  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-04.png" | absolute_url}}){: .center-image }

실제로 열어본 파일의 내용은 참담했습니다. 정상적으로 실행될수 없는 파일 구조와 중복된 라인, 아예 비어있는 파일도 있었습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-05.png" | absolute_url}}){: .center-image }

그래서 피드백을 전달했고, 다른 방식으로 작업이 가능한지 물어보았습니다. 실제로 파일이 너무 길어서 작업이 쉽지 않았다고 솔직하게 답변해주었습니다. 이번에는 기존 접근과는 다른 방식으로, 보다 실질적인 리팩토링 전략을 제시하겠다고 하니, 어떤 방식인지 한번 살펴보려고 합니다.    
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-06.png" | absolute_url}}){: .center-image }

이번에 제시하는 내용은 좀 더 상세하게 단계별로 어떻게 리팩토링을 진행해야 하는지 예시와, 그 내용을 알려줬습니다.  
- 1단계: 기능 영역 명획화 및 디렉터리 설계
- 2단계: 리팩토링 방식 예시
- 3단계: 전역 상태 분리
- 4단계: 모듈 간 의존성 줄이기
- 5단계: 모듈 번들링 방식 선택  
각 단계별로 어떻게 진행해야 하는지 예시를 설명하고 마지막으로 다음 작업을 바로 수행할 수 있다고 제안 했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-07.png" | absolute_url}}){: .center-image }

그래서 저는 바로 다음 작업을 수행하라고 했고...  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-08.png" | absolute_url}}){: .center-image }

설명만 봤을땐 완벽하게 정리된 파일인것 같지만... 역시나 파일을 다운받아 열어보니  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-09.png" | absolute_url}}){: .center-image }

... ChatGPT를 활용한 리팩토링은 역시 한계가 있을 수밖에 없겠다는 생각이 들었습니다. 1800줄이 넘는 방대한 코드이다 보니, 대화형 프롬프트만으로는 구조를 세분화하거나 실제로 파일을 나눠 정리하는 데에 어려움이 있었습니다. 보다 효율적이고 인터랙티브한 방식으로 리팩토링을 진행해보고자, 이후에는 Cursor AI를 활용하기로 결정하였습니다.  
  &nbsp;

## CursorAI
실제 코드 파일 수준에서 리팩토링을 도와줄 수 있는 도구를 찾던 중 Cursor AI를 사용해보기로 하였습니다. Cursor는 코드를 작성하는 환경 자체에서 AI가 직접 기능별 분리를 제안하거나, 파일을 자동으로 생성해주는 기능이 있어 훨씬 더 실질적인 도움이 될 것이라 판단하였습니다. 프로젝트를 Cursor에 불러온 뒤, 긴 파일을 열고 특정 함수나 섹션별로 의미 있는 구조를 만들 수 있도록 AI에게 리팩토링을 요청하였습니다. 놀라웠던 점은 단순히 코드 스타일을 정리해주는 수준을 넘어서, 어떤 부분을 어떤 파일로 분리하면 좋은지에 대해 컨텍스트를 고려한 제안을 받을 수 있었다는 점입니다. 코드의 흐름을 유지하면서도 유지보수가 쉬운 구조로 개편할 수 있었고, 이 과정에서 자동으로 생성된 파일들과 명확한 역할 분담은 매우 큰 도움이 되었습니다.  

Cursor AI를 사용하기 위해서는 먼저 공식 웹사이트(https://www.cursor.sh/)를 통해 설치 파일을 내려받아야 합니다. Cursor는 Visual Studio Code를 기반으로 한 AI 개발 에디터로, 설치 과정도 VS Code와 매우 유사하게 구성되어 있습니다. 운영체제에 맞는 설치 파일을 다운로드한 뒤 실행하면 간편하게 설치를 완료할 수 있습니다.  

설치가 완료되면, 일반 코드 에디터처럼 프로젝트 폴더를 열 수 있으며, 상단 메뉴나 사이드바를 통해 AI 기능을 사용할 수 있습니다. 특정 파일을 열고 우측 상단에 위치한 Ask AI 버튼을 누르거나, 마우스 우클릭 후 "Ask Cursor" 기능을 선택하면 코드 분석, 리팩토링, 요약 등 다양한 작업을 AI에게 요청할 수 있습니다. 저는 실제 인터랙티브한 작업이 필요하므로, Agent 기능을 사용해보겠습니다.  

처음 실행 시에는 GitHub 계정이나 Google 계정을 통해 로그인해야 하며, 무료 요금제로도 기본적인 기능을 충분히 사용할 수 있습니다. 보다 강력한 기능(예: 전체 프로젝트 구조 리팩토링, 긴 문맥 처리 등)을 사용하고 싶다면 유료 플랜을 선택할 수도 있습니다. 저는 테스트 용도로 진행하기에, 무료 플랜에서 제공하는 월150회 제한 리퀘스트로 충분할것이라 판단했습니다.  

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-10.png" | absolute_url}}){: .center-image }

앞서 설명했듯이, Cursor AI는 VS Code 기반의 툴로, 프로젝트 폴더와 대화형 프롬프트를 활용해 파일을 직접 다루며 코드 분석과 수정을 주도적으로 진행할 수 있습니다. 저는 리팩토링이 필요한 부분의 폴더를 프로젝트에 추가했고, Cursor는 해당 폴더 내 파일들을 스캔해 자동으로 인덱싱을 시작했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-11.png" | absolute_url}}){: .center-image }

코드베이스 인덱싱은 Cursor가 코드에 접근할 때, 다른 파일들과의 관계를 고려하기 위해 임베딩과 메타데이터를 저장하는 작업입니다. 코드가 수정되거나 파일이 변경되면 인덱싱이 실시간으로 다시 실행돼, 항상 최신 상태의 맥락을 유지할 수 있게 도와줍니다. 이제 본격적으로 Cursor AI를 사용해 ChatbotLlm.js 파일을 리팩토링해보겠습니다.  
  &nbsp;

## 리팩토링
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-12.png" | absolute_url}}){: .center-image }

Cursor에게 ChatGPT와 했던 똑같은 의도의 질문을 해보겠습니다. 비슷한 의도의 답변을 내놓았지만, 다른점은 Cursor는 실제로 ChatbotLlm.js에서 어떤 함수가 어떤 파일에 분리되어 들어가야 하는지를 명확하게 인지하고 제시했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-13.png" | absolute_url}}){: .center-image }

저는 ChatbotLlm.js 파일에 대해 별다른 사전 지식을 입력하지 않았지만, Cursor는 각 생성되는 파일이 어떤 역할을 해야 하는지, 어떤 파일이 새로 만들어질지, 그리고 거기에 어떤 함수가 포함돼야 하는지를 스스로 판단해 알려줬습니다. 이전에 ChatGPT와 리팩토링 테스트를 진행하면서 느꼈던 점은, 파일을 직접 수정하고 구조를 바꾸기 위해선 한 번에 모든 명령을 내리기보다는, 점진적으로 단계를 나눠서 진행하는 게 중요하다는 것이었습니다. 그래서 이번에는 Cursor가 제안한 리팩토링 전략을 그대로 따라가며, 단계별로 차근차근 진행하기로 했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-14.png" | absolute_url}}){: .center-image }
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-15.png" | absolute_url}}){: .center-image }

Cursor의 에이젼트는 항상 작업이 마무리되면, 다음 단계를 진행할지 물어보고 수행했습니다. 이제 코드 이동을 위한 함수가 준비됐으니, 실제 코드 분리/이동도 요청해보겠습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-16.png" | absolute_url}}){: .center-image }
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-17.png" | absolute_url}}){: .center-image }

또한 작업을 진행하면서 어떤식으로 요청해야 하는지도 요구했습니다. 진행방식으로 각 카테고리 별로 코드를 옮기고 정상 동작을 확인하는 프로세스를 추천했습니다. 
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-18.png" | absolute_url}}){: .center-image }

실제로 Cursor는 기존 파일을 참조해 분리된 JS 파일에 해당 함수 코드를 작성했고, 작업이 완료되면 다음 단계로 넘어갈 수 있도록 안내했습니다. 흥미로운 점은, 예를 들어 1단계가 완료됐다고 하면서 "공통 함수를 이동했다"고 알려줬지만, 실제로 chatHistory.js 파일을 확인해보면 textToLink 함수 하나만 작성돼 있는 걸 확인할 수 있었습니다. 이런 부분에서의 차이와 아쉬움은 아래에서 좀 더 자세히 다뤄보겠습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-19.png" | absolute_url}}){: .center-image }

이런 식으로 각 단계별로 옮길 파일명과 항목을 제시하면, Cursor는 실제로 해당 코드를 새 파일로 옮겨 작성해줬습니다. 동시에 남은 작업 항목들도 자동으로 현행화해 주면서 진행 상황을 시각적으로 보여줬습니다. 그런데 여기까지 작업을 마친 뒤, 실제로 수정된 파일을 직접 열어보니 함수 하나만 옮겨져 있었고, 나머지는 작업한 것처럼 표시만 돼 있었습니다. 이때 Cursor가 작업을 완료했다고 알려줬던 내용 중 일부는 실제로 반영되지 않은 것을 보고, 환각 증상이 발생했다는 걸 알아챌 수 있었습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-20.png" | absolute_url}}){: .center-image }

다시 요청했더니 이번엔 제대로 함수를 이동시켜줬습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-21.png" | absolute_url}}){: .center-image }

이제 모든 항목의 함수들이 각각 분리된 JS 파일에 모두 작성됐습니다. 다음 단계는 chatbotLlm.js에서 기존에 정의했던 함수들을, 새로 분리된 JS 파일의 함수들로 호출하도록 수정하는 작업입니다.   
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-22.png" | absolute_url}}){: .center-image }

각 모듈에 정의된 함수를 호출할 수 있도록, 호출부의 코드를 수정했습니다. 이 과정에서 chatbotMessageUI.js 파일에 주석만 작성된 함수가 여전히 남아 있는 것을 확인했고, 해당 함수를 실제로 작성하도록 지시했습니다. 이후 이 요구에 대해 여러 차례 요청을 보냈지만, 여전히 함수는 작성되지 않은 채 주석만 남아 있었고, Cursor는 함수가 다 작성됐다고 계속해서 환각 증세를 보였습니다. 결국 요청문구를 수정해, 해당 파일을 다시 전체적으로 분석한 뒤 비어 있는 함수들을 작성하도록 명확히 지시했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-23.png" | absolute_url}}){: .center-image }

이때 눈에 띄는 점은, 이전에 지시했던 네임스페이스 방식의 호출 수정을 적용하기 위해 Cursor가 스스로 코드를 옮기며 수정했다는 점입니다. 이렇게 해서 모든 파일 구성이 마무리됐고, 테스트를 진행해보았습니다. 물론 한 번에 완벽하게 동작했으면 좋았겠지만, 실제로는 화면이 의도한 대로 동작하지 않았고 에러가 발생했습니다. 이제 이 에러 내용을 그대로 Cursor에게 전달해, 어떤 부분을 어떻게 수정해야 하는지 알아보겠습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-24.png" | absolute_url}}){: .center-image }

에러 내용을 Cursor에게 전달하자, 곧바로 해당 에러를 분석하고 원인을 설명해줬습니다. 여기서 특히 흥미로웠던 점은, 왜 그 에러가 발생했는지를 지금까지 Cursor가 직접 수행한 작업 내용과 연관지어 설명했다는 점입니다. 단순히 에러 메시지를 해석하는 수준이 아니라, 자신이 어떤 코드를 어떻게 변경했는지의 흐름을 기억하고 그 결과로 인해 발생한 문제라는 인과관계를 명확히 짚어냈다는 점이 인상적이었습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-25.png" | absolute_url}}){: .center-image }
  
이어서 Cursor는 에러에 대한 해결 방법도 함께 제시해줬고, 해당 코드 블록에 있는 Apply 버튼을 눌러 바로 수정사항을 코드에 반영할 수 있었습니다. 즉, 오류를 수정하기 위해 직접 파일 위치로 이동해 타이핑할 필요 없이, 마우스 클릭 한 번으로 바로 코드를 수정할 수 있었던 점이 매우 편리했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-26.png" | absolute_url}}){: .center-image }

이 에러는 이동한 함수의 네임스페이스의 호출 문제로 발생한 에러입니다. 에러내용을 보면 아시겠지만, 위 작업중에 네임스페이스 호출 수정 작업이 정상적으로 잘 진행됐다면 발생하지 않았어야 할 에러입니다. 이 에러가 발생했다는건, 위에서 Cursor가 모든 코드를 수정했다고 말했지만, 사실은 작업을 제대로 진행하지 않은게 원인입니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-27.png" | absolute_url}}){: .center-image }

원인은 위와 같습니다. Cursor는 작업을 진행하면서 모든 프로세스를 완벽하게 수행하지 않고, 일부를 생략한 채 넘어가는 경우가 있습니다. 즉, 전체 코드를 수정하라고 지시하더라도, Cursor가 자체적으로 판단해 일부 코드는 생략하거나 사용자가 직접 수정하길 권하는 경우가 있습니다. 그렇기 때문에 Cursor를 사용해 코드를 수정한 뒤에는, 반드시 직접 결과를 확인해 의도한 대로 기능이 동작하는지 검증해야 합니다. 이러한 점은 Cursor의 프롬프트에서도 계속해서 강조하고 있습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-28.png" | absolute_url}}){: .center-image }
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-29.png" | absolute_url}}){: .center-image }

그래서 다시 작업한 내용에 대해 검토를 요청했고, Cursor에이전트가 파일들을 직접 읽으면서 검색을 수행했습니다. 그리고 어느 부분에서 호출부를 수정해야하는지 알려줬습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-30.png" | absolute_url}}){: .center-image }

다음으로 chatbotLlm.js에서 사용되던 전역변수를, 이동된 JS모듈들에서도 사용할 수 있게 변경을 요청했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-31.png" | absolute_url}}){: .center-image }

또한 에러가 발생할 경우, 역시 해당 에러가 발생한 이유를 논리적으로 설명해서 수정하는 방법을 알려줬습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-32.png" | absolute_url}}){: .center-image }

특정 변수명이나 함수명을 명시하지 않아도, 어떤 동작에 필요한 요소들을 설명만 해주면, Cursor는 자동으로 해당 코드(chatbotLlm.js의 1066~1069번째 줄)를 참조해 작업을 수행했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-33.png" | absolute_url}}){: .center-image width="900px"}
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-34.png" | absolute_url}}){: .center-image width="900px"}

다음 단계로, chatbotLlm.js에 정의돼 있던 기존 함수들을 전부 삭제하기로 했습니다. 이제 해당 함수들은 분리된 JS 모듈에서 import해 사용하므로, 코드의 가독성과 관리 편의를 위해 원본에서 제거하는 것이 좋다고 판단했습니다. 우선 삭제 대상 함수 목록을 요청해 Cursor에게 학습시키고, 그 뒤에 순차적으로 삭제 작업을 진행했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-35.png" | absolute_url}}){: .center-image }

각 항목별로 순차적으로 삭제요청을 하면, chatbotLlm.js에서 라인별로 해당 파일을 스캔하면서 함수가 있는지 판별하고 실제로 코드를 삭제시킵니다.  
  &nbsp;

## 구조 변경
Cursor를 통해 코드 리팩토링을 마친 후, 전체적인 흐름이 훨씬 깔끔해지고 가독성이 높아졌다는 점에서 큰 만족감을 느꼈습니다. 그런데 코드를 정돈하고 나니, 자연스럽게 파일이 위치한 폴더 구조에도 아쉬움이 보이기 시작하였습니다. 기능별로 잘 분리된 코드들이 여전히 하나의 폴더에 뒤섞여 있는 상태였기 때문에, 유지보수나 협업 측면에서도 비효율적이라고 판단하였습니다. 이에 따라 이번 기회에 아예 폴더 구조도 함께 정비하기로 마음먹었습니다. 공통 컴포넌트나 유틸리티 함수들을 별도로 분리하여 보다 직관적이고 관리하기 쉬운 프로젝트 구조를 만드는 것을 목표로 삼았습니다. 물론, 해당 내용도 Cursor에게 지시해보겠습니다다.  

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-36.png" | absolute_url}}){: .center-image }

이번에는 지시를 순차적으로 넘버링해서, 한번에 요구해봤습니다. 그러자 Cursor에이젼트가 실제로 파일구조에 접근하기 위해, 스크립트 코드를 실행시켰고, 요청한 동작을 수행할지에 대해 판단을 대기했습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-37.png" | absolute_url}}){: .center-image }

(폴더생성, 파일이동, 이름변경 등 실행)  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-38.png" | absolute_url}}){: .center-image }

그리고 구조가 변경이 완료되어, 각 JS파일에서 실제 코드에서 import하는 상대경로 부분도 자동으로 수정해줬습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-39.png" | absolute_url}}){: .center-image }

이때 원래 구조는 chatbotLlm.js가 메인 엔트리 역할을 하고 다른 모든 JS모듈을 import하는 형태였는데, 상위 chatbotMain.js를 메인 엔트리 구조로 변경하는 것이 좋을 것 같아 검토해봤습니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-40.png" | absolute_url}}){: .center-image }

이제 리팩토링 완료, 기능 테스트도 모두 완료했습니다. 최종적으로 Cursor에게 해당 프로젝트에서 더 수정할 사항이 있는지 요청해봅니다.  
  &nbsp;

![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-41.png" | absolute_url}}){: .center-image width="900px" }
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-42.png" | absolute_url}}){: .center-image width="900px" }
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-43.png" | absolute_url}}){: .center-image width="900px" }
![alt]({{ "/assets/2025-05-29-cursorai-refactoring/2025-05-29-cursorai-refactoring-44.png" | absolute_url}}){: .center-image width="900px" }

최종적으로 기능 중심의 구조로 개편되어, 역할에 따라 폴더가 분리되었고, 각 파일은 하나의 책임만을 갖도록 정리되었습니다. 리팩토링 결과, 코드의 가독성과 유지보수성이 크게 향상되었으며, 신규 기능 추가 시에도 영향 범위를 명확히 파악할 수 있게 되었습니다. 다만 여전히 테스트 코드의 부재, 음성 모듈 이동 등 향후 점검 및 개선이 필요한 과제로 남아 있습니다. 이번 리팩토링은 단순히 코드를 정리하는 데에 그치지 않고, 전체 프로젝트의 구조와 방향성을 재정비하는 계기가 되었습니다.

## 후기
chatbotLlm.js는 120줄의 JS코드로 줄어들었고, 각 역할을 가진 모듈JS파일을 생성해서 구조화 할 수 있었습니다. 이번 리팩토링을 통해 Cursor AI를 사용하면서 가장 인상 깊었던 점은, 코드 파일 단위에서 직접 작업을 진행할 수 있다는 새로운 방식에 대한 경험이었습니다. 이전까지는 주로 프롬프트를 통해 질문하고 응답을 받는 방식에 익숙했지만, Cursor에서는 실제 코드 편집기에서 AI와 함께 실시간으로 코드를 수정하고 구조화할 수 있다는 점이 매우 신선하게 다가왔습니다. 작업 중 대부분의 리팩토링이 프롬프트 명령으로 이뤄졌고, 저는 코드를 직접 손댈 일이 거의 없었을 정도였습니다. 또한 Cursor는 여러 AI 에이전트와 함께 연계하여 사용하면, 예컨대 설계·테스트·문서화까지 통합적으로 다룰 수 있는 훨씬 더 강력한 도구로 발전할 수 있을 것이라는 가능성도 엿볼 수 있었습니다.  

장점으로는 코드 작성 자체를 AI에게 맡길 수 있다는 점, 파일 단위 수준에서 구조적 조작이 가능하다는 점, 그리고 AI가 이전 흐름과 문맥을 기억하여 비교적 논리적인 결과를 도출한다는 점이 있었습니다. 반면, AI의 특성상 환각현상(Hallucination)이 발생할 수 있기 때문에, 실제 코드 테스트는 필수라는 점도 확인하였습니다. Cursor는 강력한 도구지만, 그 결과를 개발자가 최종적으로 검증하고 다듬는 과정은 여전히 중요하다는 것을 느끼게 되었습니다.