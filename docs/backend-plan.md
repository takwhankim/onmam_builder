# Onmam Builder Backend Plan

이 문서는 온맘닷컴 교회 홈페이지 빌더를 서비스형 CMS로 확장하기 위한 백엔드 설계 초안입니다.

현재 빌더는 단일 HTML 파일과 브라우저 로컬 저장소 중심으로 동작합니다. 이후 백엔드 작업은 현재 기능을 보존하면서 데이터 저장, 권한, 파일 업로드, 교회별 홈페이지 운영을 서버 구조로 옮기는 방향으로 진행합니다.

## 서비스 모델

온맘닷컴의 기본 모델은 교회가 처음부터 직접 사이트를 만드는 SaaS가 아니라, 온맘 제작자가 홈페이지를 제작하고 교회에는 운영에 필요한 관리자 기능을 제공하는 납품형 CMS입니다.

### 역할

#### 온맘 제작자

- 교회 사이트 생성
- 템플릿 선택
- 전체 디자인 구성
- 메뉴/페이지/게시판 구조 설계
- 키비주얼/섹션/레이아웃 설정
- 납품 전 검수
- 백업/복구
- 도메인 및 배포 관리

#### 교회 운영자

- 교회 기본 정보 수정
- 로고/키비주얼 이미지 수정
- 키비주얼 문구 수정
- 메뉴 추가/삭제/순서 변경
- 페이지 내용 편집
- 공지사항 작성
- 설교 영상 등록
- 사진첩 관리
- 게시판 글/댓글 관리
- 팝업 공지 관리

#### 방문자/성도

- 공개 홈페이지 열람
- 게시판 글쓰기
- 댓글 작성
- 파일 다운로드
- 필요 시 성도 로그인 후 제한 콘텐츠 접근

## 권장 기술 스택

### 장기 운영 권장

- Next.js
- PostgreSQL
- Prisma
- S3 또는 Cloudflare R2
- Auth.js 또는 자체 권한 모델

### 빠른 MVP 대안

- Next.js
- Supabase DB/Auth/Storage

MVP는 Supabase로 빠르게 시작할 수 있지만, 온맘닷컴이 장기적으로 여러 교회를 운영하는 플랫폼이 될 경우 PostgreSQL + Prisma 기반의 명확한 데이터 모델을 추천합니다.

## 핵심 데이터 모델

### Church

교회 단위의 최상위 고객 계정입니다.

- id
- name
- denomination
- pastorName
- address
- phone
- email
- domain
- status
- createdAt
- updatedAt

### Site

교회 홈페이지 하나의 설정 단위입니다.

- id
- churchId
- name
- theme
- design
- meta
- publishedAt
- createdAt
- updatedAt

현재 HTML 빌더의 `site` 객체가 이 모델의 시작점입니다.

### Menu

GNB와 하위 메뉴 구조입니다.

- id
- siteId
- parentId
- label
- type
- targetId
- externalUrl
- order
- visible
- categoryBanner
- createdAt
- updatedAt

`categoryBanner`에는 서브메뉴 화면 상단 이미지, 문구, 높이, 정렬, 글자 크기 등을 저장합니다.

### Page

운영자가 편집 가능한 일반 페이지입니다.

- id
- siteId
- title
- bodyHtml
- status
- createdAt
- updatedAt

### Board

공지, 자유게시판, 자료실, 사진게시판, 영상게시판 같은 게시판 설정입니다.

- id
- siteId
- title
- boardType
- options
- createdAt
- updatedAt

### Post

게시판 글입니다.

- id
- boardId
- title
- bodyHtml
- authorName
- authorId
- thumbnailMediaId
- videoUrl
- isNotice
- isSecret
- status
- createdAt
- updatedAt

### Comment

게시판 댓글입니다.

- id
- postId
- authorName
- authorId
- body
- status
- createdAt
- updatedAt

### MediaFile

로고, 키비주얼, 게시판 이미지, 첨부파일을 관리합니다.

- id
- churchId
- siteId
- uploaderId
- fileName
- mimeType
- size
- url
- storageKey
- altText
- createdAt

### User

온맘 제작자, 교회 운영자, 성도 계정입니다.

- id
- churchId
- email
- name
- passwordHash
- role
- status
- createdAt
- updatedAt

### Role / Permission

초기에는 단순 role enum으로 시작하고, 이후 세부 권한 테이블로 확장할 수 있습니다.

기본 role:

- onmam_owner
- onmam_staff
- church_admin
- church_editor
- member
- visitor

## API 초안

### 인증

- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET /api/auth/me`

### 교회/사이트

- `GET /api/churches`
- `POST /api/churches`
- `GET /api/churches/:churchId`
- `PATCH /api/churches/:churchId`
- `GET /api/sites/:siteId`
- `PATCH /api/sites/:siteId`
- `POST /api/sites/:siteId/publish`

### 메뉴

- `GET /api/sites/:siteId/menus`
- `PUT /api/sites/:siteId/menus`

메뉴는 중첩 구조 전체를 저장하는 방식으로 시작하는 것이 현재 빌더 구조와 잘 맞습니다.

### 페이지

- `GET /api/sites/:siteId/pages`
- `POST /api/sites/:siteId/pages`
- `GET /api/pages/:pageId`
- `PATCH /api/pages/:pageId`
- `DELETE /api/pages/:pageId`

### 게시판

- `GET /api/sites/:siteId/boards`
- `POST /api/sites/:siteId/boards`
- `GET /api/boards/:boardId`
- `PATCH /api/boards/:boardId`
- `DELETE /api/boards/:boardId`

### 게시글

- `GET /api/boards/:boardId/posts`
- `POST /api/boards/:boardId/posts`
- `GET /api/posts/:postId`
- `PATCH /api/posts/:postId`
- `DELETE /api/posts/:postId`

### 댓글

- `GET /api/posts/:postId/comments`
- `POST /api/posts/:postId/comments`
- `DELETE /api/comments/:commentId`

### 업로드

- `POST /api/uploads`
- `DELETE /api/uploads/:mediaId`

서버는 업로드된 파일을 S3/R2/Supabase Storage에 저장하고, DB에는 `MediaFile` 레코드를 남깁니다.

## 현재 빌더 데이터와 DB 매핑

현재 HTML 내부의 `site` 객체는 초기에는 그대로 JSON으로 저장할 수 있습니다.

1차 백엔드에서는 `Site.snapshotJson` 같은 필드에 전체 데이터를 저장합니다.
2차 백엔드에서는 메뉴, 페이지, 게시판, 게시글을 정규화된 테이블로 분리합니다.

권장 단계:

1. 현재 `site` 객체를 통째로 저장/불러오기
2. 파일 업로드만 서버로 먼저 분리
3. 게시판 글 저장을 DB로 분리
4. 메뉴/페이지/디자인을 테이블로 분리
5. 교회별 권한과 공개 사이트 렌더링 분리

## 작업 원칙

- 현재 HTML 기능을 먼저 보존합니다.
- 디자인 교체는 데이터 구조를 깨지 않는 범위에서 진행합니다.
- localStorage 저장 코드는 나중에 API 저장으로 교체하기 쉽게 유지합니다.
- 제작자 모드, 교회 운영자 모드, 방문자 화면의 역할을 계속 분리합니다.
- 게시판, 파일 업로드, AI 기능은 서버 API로 분리할 수 있도록 경계를 둡니다.

