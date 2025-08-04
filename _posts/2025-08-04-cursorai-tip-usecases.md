---
layout: post
date: 2025-08-04
title: "CursorAI 활용법, 실제 적용사례 소개"
published: true
lang: ko
excerpt: AI 에이전트 툴을 사용할때 활용한 팁, 실제 사례 소개
tags: ai
author: seounghyun
---

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-01.png" | absolute_url}}){: .center-image height="370px" }

저는 최근 프로젝트에서 CursorAI를 활용하면서 개발 생산성이 크게 향상되는 경험을 하고있습니다. 특히 레거시 코드를 분석하고 새로운 기능을 추가하는 과정에서 CursorAI의 코드 자동 완성과 리팩토링 제안 기능이 큰 도움이 되었습니다. 복잡한 비즈니스 로직을 구현할 때도 CursorAI가 제안하는 코드 패턴과 최적화 방안을 참고하여 효율적으로 개발을 진행할 수 있었습니다. 실제로 기존에 며칠이 걸리던 작업들을 몇 시간 만에 해결하는 경우도 있었으며, 코드 품질도 한층 개선되는 것을 경험했습니다.

이번 글에서는 제가 실제 프로젝트에서 CursorAI를 활용한 구체적인 사례들과 적용 방법을 상세히 소개하고자 합니다. 특히 레거시 코드 분석, 새로운 기능 개발, 버그 수정 과정에서 CursorAI를 어떻게 효과적으로 활용했는지, 그리고 이를 통해 어떤 이점을 얻을 수 있었는지를 실제 코드 예제와 함께 설명드리겠습니다. 이를 통해 여러분들도 CursorAI를 프로젝트에 도입하실 때 참고하실 수 있는 실질적인 가이드가 되었으면 합니다.

## Rules & Memories
커서의 세팅 옵션중에는 Rules & Memories 탭이 있습니다. 해당 탭의 옵션들은 커서가 동작하면서 어떤 규칙으로 계획을 세우고 답변을 생성할지, 코드를 작성하는 기준이 됩니다. 커서의 룰에는 User Rules, Project Rules이 있습니다. User Rules은 전체 프로젝트에 대해 참조하는 전역 룰을 작성하고 일반 텍스트로 전송됩니다. User Rules은 커뮤니케이션 스타일이나, 코딩 규칙을 설정하는데 적합합니다. Project Rules은 현재 열린 프로젝트의 컨벤션에 대해 에이전트가 이해할 수 있게 추가하는 룰입니다. Project Rules은 별도의 파일로 생성해 해당 파일에 스크립트가 작성됩니다. 해당 파일에 경로 패턴을 사용해서 범위를 지정하거나, 수동으로 호출할 수 있습니다. Project Rules은 코드베이스에 해당 프로젝트의 도메인 지식을 인코딩하거나, 프로젝트별로 워크플로 또는 템플릿을 자동화 할때 유용합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-02.png" | absolute_url}}){: .center-image }

제가 사용중인 User Rules입니다. 각 규칙의 뜻과 사용하는 이유를 설명드리겠습니다.  

**Write all user messages and responses in Korean** (모든 사용자 메세지와 응답은 한국어로 진행합니다)
- 커서와의 Chat 진행을 한국어로 할 수 있게 해줍니다.  

**Use English for variable and method names, Korean for comments and documentation** (변수명과 메소드명은 영어로 작성하고, 주석과 문서는 한국어로 작성합니다)
- 커서가 작성하는 코드는 영어로 작성하게 하고, 자동으로 주석을 생성할때는 한국어로 생성시킵니다.  

**When writing new code, refer to the existing codebase when determining package structures or creating new files, and if possible, write the code within the corresponding existing files** (새로운 코드를 작성할 때는 기존 코드베이스의 패키지 구조와 파일 구성을 참고하며, 가능한 경우 관련된 기존 파일 내에 코드를 작성합니다)
- 에이전트 모델이 학습한 코드스타일을 따르지 않고 기존 코드와의 컨벤션을 유지하기 위해 사용합니다. 만약 커서가 프로젝트와 호환이 안되는 최신 레퍼런스를 참고하거나 기존 컨벤션을 무시할 경우 유지보수 하기 어려워지기 때문입니다.  

**When creating a basic template in a new file, refer to the code in other files with similar roles to create the template, if possible** (새로운 파일에 기본 템플릿을 만들 때는 가능한 비슷한 역할을 하는 다른 파일들의 코드를 참고하여 템플릿을 작성합니다)
- 화면 레이아웃이나, 백엔드 프로세스 코드를 작성할때 다른 패키지 구조의 코드와 통일성을 갖게 하기 위해 사용합니다.  

다음으로 Memories 기능이 있습니다. Memories기능은 채팅 기반으로 자동으로 현재 프로젝트에 필요한 룰을 제안 및 생성합니다. 예를 들어, 만약 채팅 진행중에 새로운 룰이 필요할 것 같다고 판단되면, 커서는 자동으로 해당 룰을 생성해서 Memories에 추가할지 물어봅니다. 해당 항목들은 Ruels처럼 별도의 옵션으로 관리되어, 답변 생성을 위한 컨텍스트에 포함됩니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-03.png" | absolute_url}}){: .center-image }

해당 메모리는 커서가 저와의 대화중에 해당 규칙이 필요하겠다고 판단해서 자동으로 추가된(제안한) 규칙들입니다.

**The user prefers that the assistant ask for confirmation before moving forward with code implementation steps.** (사용자는 코드 구현 단계를 진행하기 전에 사용자에게 확인을 받고 진행하는 것을 선호합니다)
- 제가 에이전트 모드로 항상 작업전에 확인을 받는 절차를 요구했더니 추가됐습니다.  

**User prefers the assistant to always follow the current project's conventions over general best practices when making decisions.** (사용자는 의사 결정 시 일반적인 모범 사례보다 현재 프로젝트의 규칙을 우선적으로 따르는 것을 선호합니다)
- 커서가 이상적인 코드를 작성했지만 저는 기존 규칙을 유지하려고 하자 추가됐습니다.  

**The user prefers to handle date types (timestamp columns) as String in domain classes.** (도메인 클래스에서 날짜 타입(timestamp 컬럼)을 String으로 처리하는 것을 선호합니다.)
- 날짜 타입을 항상 String으로 변경해달라고 요구하자 추가됐습니다.  

이렇게 Memories 기능을 활성화 하면, 커서는 자동으로 제가 수정요청하는 작업에 대해 규칙성을 찾아내어 Memories에 추가하고 다음 답변에 해당 기억을 반영합니다. 즉 필요하다면 Ruels를 추가할 수도 있지만 이렇게 자동으로 추가되기도 합니다.

## Ctrl + K, Ctrl + L
항상 모든 작업을 채팅으로 진행하기 보다 가끔은 빠르게 인라인 편집기로 수정하고 싶을 때도 있습니다.
특정 함수의 기능을 수정한다거나, 변수 명칭을 통일성있게 바꾸거나, 선택한 부분의 코드를 모듈화 하는 등의 작업입니다.
코드를 커서로 잡고 Ctrl+K를 입력하면 선택 항목 편집 모드가 됩니다. 그리고 원하는 지시를 입력하면 해당 선택한 코드에 대해서만 작업을 진행합니다.
만약 코드를 선택안하고 Ctrl+K를 입력하면 커서 위치에 새 코드를 입력하고, 이때 전체 맥락을 파악하기 위해 관련 주변 코드를 자동으로 포함시킵니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-04.png" | absolute_url}}){: .center-image }

선택 항목 편집에서 Edit 모드를 변경할 수도 있습니다.
1. Edit Selection : 기본 모드로 선택한 코드에 대해 지시를 수행합니다.
2. Edit Full File : Ctrl+Shift+Enter 입력시 파일 전체 변경 모드로 바뀝니다. 해당 모드에서 사용자의 지시는 현재 열려있는 파일 전체를 대상으로 합니다.
3. Quick Question : 간단한 질문 모드는 선택한 코드에 대해 대화 모드로 바뀝니다. 해당 모드는 코드를 수정하진 않지만 선택한 코드에 대해 어떤 역할을 하는지, 개선방안은 있는지등의 분석을 요청할 수 있습니다.
4. Send to Chat : 선택한 코드를 채팅으로 전송시킵니다. 채팅 탭에서는 다중 파일 편집, 상세 설명, 고급AI기능을 제공할 수 있습니다.

이렇게 인라인 편집 기능을 사용해서
1. 채팅 모드로 전체적인 코드 맥락 파악 및 작업 지시 > 
2. 생성된 코드에 대해 인라인 편집으로 빠른 수정 > 
3. 3.자세한 수정이 필요한 부분은 채팅으로 고급기능을 활용
순서로 작업을 진행할 수 있습니다.

**<Ctrl+L 로 선택한 코드 동작 수정>**  
추천 질문 목록 조회 및 UI를 처리하는 getQuestionList() 함수를 전체 선택한 뒤, Ctrl+L로 채팅으로 보내서 필요한 코드 수정을 합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-05.png" | absolute_url}}){: .center-image }

getQuestionList는 관리자화면에서 등록된 질문목록을 조회해서 화면 우측영역에 카드형태로 표시하는 함수입니다.
해당 함수에서 우측영역(side)인 데이터의 경우 랜덤하게 5개만 표출시키도록 코드를 수정하라고 지시해보겠습니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-06.png" | absolute_url}}){: .center-image }

그러면 커서가 원하는 의도대로 함수의 동작을 수정하고 , 테스트 해보면 실제로 코드가 잘 동작하는 것을 확인할 수 있습니다.

## Tab
Tab은 커서에서 제공하는 자동 완성 기능입니다. 커서가 제공하는 Tab 제안에 대해 사용자가 수용하거나, Esc버튼으로 거부하는 방식으로 의도를 주입할수록 더욱 효과적으로 작동합니다. 텍스트를 추가하거나 코드에 커서를 두면 자동으로 반투명한 제안 코드가 표시되거나, 코드를 선택하면 바로 옆에 diff 팝업이 표시됩니다. Tab은 코드베이스 기반으로 작동하기 때문에, 해당 코드가 어떻게 수정되거나 작성돼야 할지 알고 있습니다. 또한 자동완성 기능 외에 더 추가적인 기능도 있습니다. 예를 들어, 변수명을 수정할 경우 해당 변수가 사용된 라인으로 이동을 제시 하고, 해당 변수를 수정한 변수명으로 교체할지 제안합니다.

- Tab은 자동으로 파일에서 다음 편집 위치를 예측하고 이동할 위치를 제안합니다.
![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-07.png" | absolute_url}}){: .center-image }

html 레이아웃에서 '메인(좌측)'의 포맷으로 표시된 텍스트를 '메인-좌측' 으로 변경하니 해당 템플릿에서 똑같은 형태를 하고있는 곳의 라인으로 이동을 제안합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-08.png" | absolute_url}}){: .center-image }

tab 키를 누른 후 해당 라인으로 이동하면 '메인(중앙)'을 '메인-중앙' 으로 변경할 것을 제안합니다. tab을 누르면 자동으로 텍스트가 바뀌고 다음 Tab 동작을 표시합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-09.png" | absolute_url}}){: .center-image }

다시 tab을 누르면 메인(우측) 으로 표시되는 텍스트로 찾아가서 메인-우측으로 수정을 제안합니다. 즉 똑같은 형식으로 표시되는 텍스트를 추가 조작없이 tab만 눌러서 원하는 포맷으로 변경했습니다.

- Tab은 여러 파일간의 컨텍스트 편집을 예측하고 자동으로 파일간 이동을 제안합니다.
![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-10.png" | absolute_url}}){: .center-image }

서비스의 인터페이스 참조 코드를 deleteChatbotQuestion 에서 deleteChatbotQuestionConfirm 으로 변경했습니다. 커서는 해당 인터페이스가 정의된 서비스 자바 파일로 이동을 제안합니다.

tab을 누르면 해당 파일이 열리고, 자동으로 인터페이스가 정의된 라인으로 커서가 이동합니다.
![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-11.png" | absolute_url}}){: .center-image }

정의된 인터페이스 명칭을 사용하는 곳에서 수정한것처럼 deleteChatbotQuestionConfirm으로 수정할 것을 제안합니다.

이렇게 서로 다른 파일이더라도, 컨텍스트를 인식해서 다른 파일에서의 편집도 지원합니다. Tab기능은 제가 커서를 사용하면서 생산성 향상의 가장 주력으로 사용하는 기능으로, 탭 키를 입력 하는 것만으로 새로운 프로세스 로직을 정의할때 파일 이동, 라인 이동 없이 탭키 하나만으로 작업을 가능하게 해줍니다. Tab기능을 사용하면서 가장 중요한 것은, Tab은 전후 사용자 동작의 의도를 파악해서 제안을 하기 때문에 원하는 작업을 하기 위해서 해당 제안을 유도하는 방식의 사용법 체득입니다.

**< Tab 기능 사용 예시 >**  
새로운 서비스 인터페이스를 정의 => (메모리)  
서비스 인터페이스 파일에서 탭으로 해당 인터페이스 자동 생성 => (메모리)  
인터페이스 구현체 파일에서 탭으로 해당 메소드 구현 => (메모리)  
해당 서비스에서 탭으로 DAO 매퍼 구현 => (메모리)  
매퍼 XML 파일에서 탭으로 해당 쿼리 구현

- 주석을 미리 작성하고 해당 코드를 Tab으로 생성  
이 방법은 제가 화면에서 동작하는 이벤트 스크립트를 생성할때 자주 사용하는 방법입니다. 주석으로 원하는 동작을 미리 작성하고, 다음 라인에서 탭을 사용하면 자동으로 제가 원하는 동작을 구현하는 스크립트를 제안합니다. 해당 방식으로 대화형 입력을 수행 안하고 바로 코드를 생성시킬 수 있습니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-12.png" | absolute_url}}){: .center-image }

모달 닫기 기능을 구현중에 ESC 버튼을 클릭하면 모달이 닫히도록 이벤트를 추가하기 위해 기존 코드 아래에 주석을 작성 합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-13.png" | absolute_url}}){: .center-image }

document... 같이 기존 정의된 이벤트와 똑같은 컨벤션으로 코드를 작성하기 시작하면 Tab이 자동으로 다음 작성 코드를 표시합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-14.png" | absolute_url}}){: .center-image }
 

여기서 tab을 누르면 이제 커서가 확실하게 의도를 파악했기 때문에 완성된 js 코드를 제안합니다. 이렇게 tab버튼을 두번 누르는것으로 새로운 js 이벤트 코드를 작성할 수 있었습니다.

## 화면을 설계하는 방법
웹 화면을 개발할때 최초 레이아웃을 구성하기 위해 Chat의 이미지 첨부기능을 사용할 수 있습니다. 구현하고자 하는 화면의 계획서를 캡쳐해서 이미지 형태로 Chat에 첨부하면, 커서는 해당 이미지를 분석해서 필요한 html 구조를 생성하고 자동으로 작성합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-15.png" | absolute_url}}){: .center-image }

캡쳐한 이미지와 코드가 작성될 jsp 파일을 컨텍스트에 포함시키고 이미지와 같은 html 을 작성해달라고 요청합니다. 이때, 기존 관리페이지의 다른 레이아웃을 참고해달라고 하면 다른 화면들에서 구성한 컴포넌트들의 스타일을 참고해서 똑같이 작성해줍니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-16.png" | absolute_url}}){: .center-image }
&nbsp;
![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-17.png" | absolute_url}}){: .center-image }


만약 생성된 html 코드에서 수정이 필요한 부분이 있다면 Chat으로 해당 수정사항을 요청할 수도 있습니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-18.png" | absolute_url}}){: .center-image }

컴포넌트 스타일을 통일하고 싶다면 Chat으로 요청하면 됩니다. 다른 관리자페이지 파일을 열어서 버튼 컴포넌트를 찾고, 해당 요소에 어떤 스타일이 적용됐는지 파악하고 그걸 다시 작업중이던 파일로 돌아와서 적용할 필요없이 에이전트가 알아서 기존 스타일을 가져와서 적용시켜줍니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-19.png" | absolute_url}}){: .center-image }

만약 특정 화면의 컴포넌트와 비교하고 똑같은 스타일을 적용하고 싶다면 해당 내용을 Chat으로 입력하면 됩니다.

커서로 화면을 만들때는 기존 화면 레이아웃이 존재한다면 해당 파일을 컨텍스트로 포함시키고 전체 틀을 구성한뒤, 요소별로 세부 수정을 진행하면 원활하게 작업할 수 있습니다.

## MCP활용
MCP(Model Context Protocol)란 인공지능 모델이 외부 데이터 및 도구에 접근할 수 있도록 하는 개방형 프로토콜입니다. 

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-20.png" | absolute_url}}){: .center-image }

설정의 Tools & Integrations 탭에서 원하는 MCP를 등록하고 관리할 수 있습니다.
필요한 MCP 서버를 직접 개발해서 사용할 수도 있지만, 다른 사람들이 개발한 다양한 MCP서버를 활용할 수도 있습니다.
저는 PostgreSQL RDBMS에 접근할 수 있게 해주는 MCP 서버를 커서에 등록해서 사용했습니다.
해당 MCP서버를 등록하면 커서는 DB서버에 접근해서 스키마 및 테이블 정보를 확인하고 분석할 수 있습니다. 다만 MCP의 경우 외부와의 접근을 허용하는 환경이 되기 때문에, 주의깊게 사용해야 합니다. 제가 등록한 postgres MCP는 Read only tool만 사용할 수 있어서, 데이터의 정합성을 지키고 변형을 방지할 수 있었습니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-21.png" | absolute_url}}){: .center-image }

특정 스키마의 테이블 정보를 확인하고, 해당 테이블에 사용할 VO 클래스의 필드를 작성해달라고 요청했습니다. 
커서는 자동으로 해당 테이블을 조회해서 필요한 필드를 추가했습니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-22.png" | absolute_url}}){: .center-image }


쿼리를 작성한 매퍼가 정상적으로 동작하지 않을때, 대상 테이블 정보를 조회해서 쿼리를 어떻게 수정해야 하는지 물어볼 수도 있습니다.
이외에도 다양한 MCP 서버가 존재하며, 필요에 따라 적절히 활용하면 에이전트의 능률을 크게 향상시킬 수 있습니다.


## 신규 도메인 추가 작업의 흐름
신규 도메인(프로세스) 기능을 추가하는 작업을 진행할때, 주로 아래와 같은 순서로 진행합니다.

1. Chat모드를 Ask모드로 변경
- Agent모드로 진행하면 바로 코드에 수정을 진행하기 때문에, 우선 맥락을 이해시키기 위해 Ask모드로 진행합니다.

2. 지금부터 진행할 작업에 대해 어떤 내용인지 간략한 설명
- 핵심 기능, 필요한 코드, 수정되어야 하는 패키지 구조등을 설명합니다.

3. 핵심 프로세스에 대한 기능 설계 제안 요청
- 요구사항을 처리하기 위해 어떻게 코드를 작성할지 요청합니다

4. 대화형태로 구현기능에 대한 구체적인 내용 논의
- 제안한 내용에 대해 대화형식으로 수정사항을 요청합니다.

5. Agent 모드로 변경후 '지금까지의 대화를 기반으로' 새로 추가되는 기능에 대해 계획을 세우고, 차례대로 실행 요청.
- 단계별로 계획을 세워서 어떤 코드를 수정해야 할지 스스로 판단하게 합니다.

7. 각 단계를 실행할때는 보고후 확인후에만 진행하도록 요청
- 한번에 모든 단계를 진행시키면 프롬프트가 오작동할 수 있으므로 단계별로 진행하고 직접 체크합니다.

**< 실제예시 >**  
*화면에서 특정 컴포넌트 클릭시 데이터의 노출 및 사용 이력을 조회하는 프로세스 및 화면 구현*


![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-23.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-24.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-25.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-26.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-27.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-28.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-29.png" | absolute_url}}){: .center-image }

대화 도중 예상 가능한 문제점이 있을 경우 해당 문제에 대한 예외처리가 고려되어야 한다는걸 인지시킵니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-30.png" | absolute_url}}){: .center-image }

(Agent모드로 변경)

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-31.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-32.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-33.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-34.png" | absolute_url}}){: .center-image }

단계별 진행도중 수정사항이 필요하면 중간에 요청합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-35.png" | absolute_url}}){: .center-image }

완성된 프로세스 코드에 대해 빌드 및 테스트 진행

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-36.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-37.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-38.png" | absolute_url}}){: .center-image }

수정사항에 파일을 맨션으로 첨부해서 명확한 작성을 요청합니다.

**< 완성된 프로세스에 대해 추가 수정요청사항 적용 >**
![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-39.png" | absolute_url}}){: .center-image }

다시 1단계로 돌아가서 Ask모드로 진행합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-40.png" | absolute_url}}){: .center-image }

추가반영되는 로직에 대해 에이전트가 잘못 인지하고 있는 사실을 명확히 설명해서 구체화 시킵니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-41.png" | absolute_url}}){: .center-image }

설계가 완성되면 Agent모드로 변경하고 계획대로 작업을 수행합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-42.png" | absolute_url}}){: .center-image }

계획 단계에서 미리 작성한 부분은 건너뛰도록 지시합니다.

![alt]({{ "/assets/2025-08-04-cursorai-tip-usecases/2025-08-04-cursorai-tip-usecases-43.png" | absolute_url}}){: .center-image }

이렇게 시스템에 새로운 기능인 '특정 테이블 라인을 클릭시 해당 정보의 이력 로그를 조회하고 표출하는 기능'을 화면단부터 백엔드 프로세스까지 전부 대화형 작업으로 진행 할 수 있었습니다. (테스트 포함 약 30분정도 소요)

지금까지 제가 경험한 CursorAI의 실제 활용 사례와 적용 방법에 대해 살펴보았습니다. CursorAI는 단순한 코드 자동 완성 도구를 넘어서, 개발자의 생각을 이해하고 함께 문제를 해결하는 든든한 페어 프로그래밍 파트너로서의 역할을 훌륭히 수행하고 있습니다. 특히 복잡한 레거시 코드를 다루거나 새로운 기능을 개발할 때, CursorAI의 지능적인 제안들이 개발 생산성과 코드 품질 향상에 큰 도움이 되었습니다. 앞으로도 AI 기술의 발전과 함께 CursorAI가 제공하는 기능들이 더욱 고도화될 것으로 기대되며, 이를 통해 개발자들의 업무 환경이 더욱 스마트해질 것으로 전망합니다.