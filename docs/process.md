# SSR 구현 과정 및 문제 해결

## ❌ 문제 상황1 : 클라이언트 코드를 서버 Node 환경에서 실행하려고 해서 발생하는 에러

### **에러 메시지**

```
node:internal/modules/esm/resolve:263
    throw new ERR_UNSUPPORTED_DIR_IMPORT(path, basePath, String(resolved));
          ^

Error [ERR_UNSUPPORTED_DIR_IMPORT]: Directory import '/Users/angielee/Desktop/hanghae/front_6th_chapter4-1/packages/vanilla/src/components' is not supported resolving ES modules imported from /Users/angielee/Desktop/hanghae/front_6th_chapter4-1/packages/vanilla/src/pages/HomePage.js
```

### **에러 원인**

클라이언트 코드 서버 실행: 브라우저용 코드를 Node.js 환경에서 실행하려고 시도

```
const html = HomePage();
```

### ✅ 해결 방법

Vite SSR 모듈 로드 방식 사용

개발 환경에서는 브라우저 전용 코드를 Node.js에서 직접 실행하면 안 되므로, Vite의 `ssrLoadModule`을 사용해 서버 전용 엔트리(`main-server.js`)를 불러온다.  
프로덕션에서는 빌드된 SSR 번들을 동적으로 import해서 사용한다.

```js
const render = async (url) => {
  try {
    if (prod) {
      // 프로덕션: 빌드된 SSR 모듈 사용
      const { render } = await import("./dist/vanilla-ssr/main-server.js");
      return await render(url);
    } else {
      // 개발: Vite SSR 모듈 로드
      const { render } = await vite.ssrLoadModule("/src/main-server.js");
      return await render(url);
    }
  } catch (error) {
    console.error("Render error:", error);
    return { html: "<div>Error</div>", head: "", initialData: {} };
  }
};
```

## ❌ 문제 상황2 : fetch is not a function

### **에러 메시지**

```
fetch is not a function
```

### **에러 원인**

```js
const getTemplate = async () => {
  if (prod) {
    return fs.readFileSync(join(__dirname, "dist/vanilla/index.html"), "utf-8");
  } else {
    return fs.readFileSync("./dist/vanilla/index.html", "utf-8");
  }
};
```

- 개발 모드에서 빌드된 HTML을 사용
- 빌드된 HTML은 프로덕션용으로 최적화되어 있음
- 개발 모드에서는 Vite의 개발 서버 기능이 필요
- 빌드된 HTML에는 개발용 스크립트가 없어서 fetch API가 제대로 로드되지 않음

### ✅ 해결 방법

```js
const getTemplate = async () => {
  if (prod) {
    // ! 프로덕션: 빌드된 HTML 사용
    return fs.readFileSync(join(__dirname, "dist/vanilla/index.html"), "utf-8");
  } else {
    // ! 개발: 개발용 HTML 사용 + Vite 변환
    // return fs.readFileSync("./dist/vanilla/index.html", "utf-8");

    let template = fs.readFileSync("./index.html", "utf-8");
    return await vite.transformIndexHtml("/*", template);
  }
};
```

개발 모드에서는 Vite의 개발 서버가 제공하는 index.html을 사용해야 한다.
이렇게 하면 Vite가 필요한 개발용 스크립트와 HMR(Hot Module Replacement) 등을 자동으로 삽입해준다.
따라서, index.html을 읽은 뒤 vite.transformIndexHtml로 변환하여 반환한다.

## 📊 현재 SSR 구현 흐름도

┌─────────────────────────────────────────────────────────────────┐
│ 클라이언트 요청 │
│ (브라우저) │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ Express 서버 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 1. JSDOM 환경 설정 │ │
│ │ setupServerJsdom() │ │
│ │ - window, document, localStorage 등 설정 │ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ 라우트 처리 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 2. URL 파싱 및 라우트 매칭 │ │
│ │ app.use("\*all", async (req, res) => { │ │
│ │ const url = req.originalUrl.replace(base, ""); │ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ Vite SSR 모듈 로드 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 3. 개발환경: vite.ssrLoadModule("/src/main-server.js") │ │
│ │ 프로덕션: import("./dist/vanilla-ssr/main-server.js")│ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ main-server.js │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 4. ServerRouter 라우트 매칭 │ │
│ │ - "/" → HomePage │ │
│ │ - "/product/:id" → ProductDetailPage │ │
│ │ - "/404" → NotFoundPage │ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ 데이터 프리페칭 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 5. 라우트별 데이터 로드 │ │
│ │ - "/": getProducts({ limit: 20 }) │ │
│ │ - "/product/:id": getProduct(params.id) │ │
│ │ - Store 상태 업데이트 │ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ HTML 렌더링 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 6. 페이지 컴포넌트 실행 │ │
│ │ - HomePage() → HTML 문자열 │ │
│ │ - ProductDetailPage() → HTML 문자열 │ │
│ │ - NotFoundPage() → HTML 문자열 │ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ 템플릿 치환 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 7. HTML 템플릿 처리 │ │
│ │ - index.html 읽기 │ │
│ │ - <!--app-html--> → 렌더링된 HTML │ │
│ │ - <!--app-head--> → head 태그 │ │
│ │ - window.**INITIAL_DATA** 스크립트 주입 │ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ 응답 전송 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 8. 완성된 HTML 응답 │ │
│ │ res.status(200).set({ "Content-Type": "text/html" })│ │
│ │ .end(finalHtml); │ │
│ └─────────────────────┬───────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│ 클라이언트 하이드레이션 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 9. 브라우저에서 JavaScript 실행 │ │
│ │ - window.**INITIAL_DATA** 복원 │ │
│ │ - Store 상태 동기화 │ │
│ │ - 이벤트 리스너 연결 │ │
│ │ - 인터랙티브 기능 활성화 │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

```

## 🔄 주요 컴포넌트별 역할

### **server.js**

- Express 서버 설정
- JSDOM 환경 초기화
- Vite 미들웨어 통합
- 라우트 핸들링 및 템플릿 치환

### **main-server.js**

- 서버사이드 라우터 구현
- 데이터 프리페칭 로직
- 페이지 컴포넌트 렌더링
- 초기 상태 데이터 수집

### **setupJsDom.js**

- Node.js 환경에서 브라우저 API 모킹
- window, document, localStorage 등 설정
- fetch API 대체 구현

### **ServerRouter**

- 정규식 기반 라우트 매칭
- 동적 파라미터 추출
- 라우트 핸들러 실행

## ⚠️ 현재 문제점

1. **Mock fetch 응답 구조**: API 응답과 일치하지 않는 구조
2. **Store 상태 관리**: 서버/클라이언트 간 상태 동기화 문제
3. **에러 처리**: 각 단계별 에러 핸들링 부족
4. **성능 최적화**: 캐싱 및 최적화 로직 부재

```

## ❌ 문제 상황3 : Prefetch error: TypeError: fetch is not a function

### **에러 메시지**

```
Prefetch error: TypeError: fetch is not a function
```

### **에러 원인**

- JSDOM 환경에서 fetch가 제대로 설정되지 않았기 때문

### ✅ 해결 방법

- setupJsDom.js에서 Node.js의 내장 fetch를 우선적으로 사용하도록 수정

```js
// Node.js의 내장 fetch 사용 (Node.js 18+)
if (typeof globalThis.fetch === "undefined") {
  // Node.js 18+ 에서는 fetch가 전역으로 사용 가능
  if (typeof fetch !== "undefined") {
    globalThis.fetch = fetch;
  } else {
    console.warn("Node.js fetch not available, using jsdom fetch");
    globalThis.fetch = dom.window.fetch;
  }
}
```

## ❌ 문제 상황4 : Prefetch error: TypeError: Failed to parse URL from /api/products?page=1&limit=20&sort=price_asc

### **에러 메시지**

```
Prefetch error: TypeError: Failed to parse URL from /api/products?page=1&limit=20&sort=price_asc
```

### **에러 원인**

> 서버 사이드에서 fetch를 사용할 때 상대 경로 URL을 파싱할 수 없기 때문

> 상대 경로의 기준점이 다름

- 클라이언트 환경: 브라우저에서 실행될 때는 현재 페이지의 URL이 기준점

예: http://localhost:5173/products 페이지에서 /api/products는 http://localhost:5173/api/products로 해석됨

- 서버 환경: Node.js 서버에서 실행될 때는 파일 시스템의 현재 작업 디렉토리가 기준점

예: /api/products는 file:///api/products로 해석되거나 에러 발생

### ✅ 해결 방법

```js
// API 기본 URL 설정
const getApiBaseUrl = async () => {
  if (import.meta.env.SSR) {
    console.log("server logic");

    // 서버 환경에서는 절대 URL 사용
    // 개발 환경: 5173, SSR 서버: 5174
    const port = process.env.PORT || 5174;

    return `http://localhost:${port}`;
  } else {
    console.log("client logic");

    // 클라이언트 환경에서는 상대 URL 사용
    return "";
  }
};
```

환경에 따라 baseURL 다르게 설정
ssr 환경에서는 절대경로 사용

> 근데도 계속 에러 발생

> Prefetch error: SyntaxError: Unexpected token '<', "<!doctype "... is not valid JSON

### **에러 원인**

- SSR 서버에 API 라우트가 구현되지 않았기 때문입니다. SSR 서버가 API 서버 역할도 하도록 수정

### ✅ 해결 방법

MSW(Mock Service Worker)를 실제로 동작하게 만듭니다
listen() 메서드가 호출되어야 MSW가 요청을 가로채기 시작합니다

```
mswServer.listen({
  onUnhandledRequest: "bypass",
});
```

MSW가 모킹하지 않은 API 요청을 만났을 때
해당 요청을 MSW가 가로채지 않고 그대로 통과시킵니다
실제 네트워크로 요청이 전달되어 실제 API 서버에 도달합니다

```
// MSW로 모킹된 API
GET /api/mocked-endpoint → MSW가 가로채서 모의 데이터 반환

// MSW로 모킹되지 않은 API
GET /api/products → MSW가 가로채지 않고 실제 서버로 전달
```
