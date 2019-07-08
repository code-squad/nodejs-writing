## 미들웨어

전 직장에서 미들웨어라는 용어를 엔터프라이즈에서 사용하는 미들웨어로 알고 있던 터라 express.js의 미들웨어와 혼동했던 적이 있었습니다. 엔터프라이즈에서의 미들웨어는 낮은 수준의 메커니즘을 추상화하는 데 도움이 되는 제품을 말합니다. 미들웨어를 도입하면 주로 OS API, 네트워크, 메모리 관리 등과 같은 기능을 제공하기 때문에 개발자는 비즈니스 로직을 구현하는 데 집중할 수 있습니다.

![2880px-Linux_kernel_and_gaming_input-output_latency svg](https://user-images.githubusercontent.com/18232901/60751995-5c424c80-9ffa-11e9-81d4-6786b7483a6c.png)

문자 그대로 다가가면 하위 서비스와 애플리케이션 사이에 위치하는 모든 종류의 소프트웨어라고도 할 수 있습니다. express.js에서의 미들웨어는 엔터프라이즈에서 사용하는 미들웨어의 의미보다 넓은 쪽이 속하는 것 같습니다. express.js의 미들웨어는 요청과 응답의 중간에 위치하여 사용자의 요청에 따른 응답을 처리하는 함수들의 집합을 의미합니다.

### 미들웨어 구조

![middleware-30b3b30ad54e21d8281719042860f3edd9fb1f40f93150233a08165d908f4631](https://user-images.githubusercontent.com/18232901/60752100-18504700-9ffc-11e9-890f-5cfe68f6cdef.png)

위의 그림처럼 미들웨어는 요청이 들어오면 등록된 미들웨어를 차례로 실행하여 사용자에게 보낼 응답을 처리합니다. 한 미들웨어에서 작업을 마치면 다음 미들웨어로 `Request` 객체와 `Response` 객체를 보내거나 `send()` 메소드로 응답을 처리하여 작업을 끝낼 수도 있습니다. 그림에는 나와있지 않지만, 미들웨어에서 작업을 처리하다가 에러가 발생하면 가장 마지막에 있는 미들웨어로 가게 됩니다. 이 미들웨어는 주로 에러를 처리하는 로직을 사용하여 사용자에게 오류가 발생했다는 사실을 알려줍니다.

### 구현

미들웨어의 구현은 [김정환님의 블로그](http://jeonghwan-kim.github.io/series/2018/12/08/node-web-8_middleware.html)에서 공부했습니다. 미들웨어들을 배열에 집어넣고, 미들웨어를 차례로 실행하는 함수를 재귀로 구현하였습니다.

```javascript
const Middleware = function() {
  const middlewareContainer = []; // 미들웨어를 담는 배열입니다.
  let _req, _res; // 원본 요청, 응답 객체를 참조합니다. 다음 미들웨어를 실행할 때 함수의 인자로 사용됩니다.

  const add = fn => {
      middlewareContainer.push(fn); // 배열에 미들웨어를 추가합니다.
  };

  const _run = (index, err) => {
    // 재귀 종료 조건은 배열 원소를 참조할 수 없는 인덱스를 가질 때입니다.
    if (index < 0 || index >= middlewareContainer.length) {
      return;
    }

    const nextMw = middlewareContainer[index];
    const next = err => _run(index + 1, err); // 다음 미들웨어를 실행하는 함수입니다. 

    // 미들웨어 실행 도중 에러가 발생했을 경우에 실행될 조건문입니다. 
    if (err) {
      const isNextErrorMw = nextMw.length === 4

      // 현재 실핼할 미들웨어의 parameter가 4개라면 에러를 처리하는
      // 미들웨어라는 뜻입니다. 이때는 미들웨어를 실행합니다.
      // 만약 4개가 아니라면 일반 미들웨어이므로, 다음 미들웨어로 넘어갑니다.
      return isNextErrorMw ?
        nextMw(err, _req, _res, next) :
        _run(index + 1, err)
    }

    // 다음 미들웨어를 실행합니다.
    nextMw(_req, _res, next);
  }

  // 외부에서 호출하게될 인터페이스입니다.
  const run = (req, res) => {
    _req = req;
    _res = res;
    _run(0);
  };

  return {
    add,
    run,
  }
};


module.exports = Middleware;
```

## 미들웨어 스타일의 라우터

아쉽게도 블로그에는 미들웨어 스타일의 라우터가 구현되어 있지 않았습니다. Node.js의 HTTP 모듈만 가지고 토이 프로젝트를 진행하고 있었는데, 미들웨어 스타일의 라우터만 도입하면 내 코드를 깔끔하게 정리할 수 있을 텐데라는 생각을 하고 있었습니다. 그래서 구현한 미들웨어를 토대로 라우터를 구현하기로 마음먹었습니다.

### express.js 라우터의 특징

최대한 비슷하게 구현하기 위해서 express.js의 예제 코드와 Github 소스를 살펴보면서 특징을 관찰했습니다. 그 결과를 4가지로 정리하면 다음과 같습니다.

1. `express.Router()`로 라우터를 import한 다음 처리할 로직을 작성하고 라우터 객체를 다시 export.
2. 라우터마다 base가 되는 경로가 있고, 내부 로직을 쓸 때는 subpath만 작성. 정규식을 쓸 수 있고 `:`으로 parameter를 가져올 수 있음.
3. `express.use()`로 basepath와 라우터를 등록해서 사용.
4. 콜백 앞에 다른 미들웨어를 끼워 넣을 수 있음.

각각의 특징에 대응하여, 구현 방향을 이렇게 잡았습니다.

1. Router 클래스를 만들어 사용.
2. 정규식과 `:` 기호까지는 구현하지 않고, basepath와 subpath까지 나누는 것만 구현.
3. 이미 구현한 Middleware 클래스의 `add()` 메소드에 라우터를 등록할 수 있도록 parameter를 1개에서 2개로 변경. Router 클래스에 `run()` 메소드를 만들고, 함수를 Middleware 클래스에 등록할 때는 이 메소드를 넣도록 구현.
4. 특정 메소드와 경로가 일치할 때 실행할 콜백 함수 앞에 여러 개의 미들웨어를 넣을 수 있는데, 이걸 중첩된 미들웨어 구조로 판단함. 따라서 Router 클래스에 Middleware 클래스의 인스턴스를 멤버로 가지도록 구현.

### 구현

#### 아키텍처

실행 흐름을 그림으로 정리하면 다음과 같습니다.

![todo-app middleware](https://user-images.githubusercontent.com/18232901/60646215-bc5bb600-9e75-11e9-8c76-a164ec153aa8.jpg)

미들웨어를 쭉 실행하다가 라우터를 만나면 라우터에 멤버로 존재하는 미들웨어를 다시 실행하도록 만들었습니다.

#### 라우터 구현

일단 코드를 먼저 적겠습니다.

```javascript
const Middleware = require('./middleware');

class Router {
  constructor() {
    this.basePath = '/';
    this.routes = {
      "GET": {},
      "POST": {},
      "PUT": {},
      "DELETE": {},
    };
  }

  add(method, path, middlewares = []){
    const routerMiddleware = new Middleware();
    middlewares.forEach(middleware => routerMiddleware.add(middleware));
    this.routes[method][path] = routerMiddleware;
  }

  get(path, ...middlewares) {
    this.add("GET", path, middlewares);
  }

  post(path, ...middlewares) {
    this.add("POST", path, middlewares);
  }

  put(path, ...middlewares) {
    this.add("PUT", path, middlewares);
  }

  delete(path, ...middlewares) {
    this.add("DELETE", path, middlewares);
  }

  find(method, url){
    const routeUrl = url.replace(this.basePath, '') || '/';
    return this.routes[method][routeUrl];
  }

  lastRouterMiddleware(err, req, res, next) {
    req.next(err);
  }

  run(req, res, next) {
    const route = this.find(req.method, req.url);
    if(req.url.split('/')[1] === this.basePath.split('/')[1] && route){
      req.next = next;
      const routerMiddlewares = route;

      routerMiddlewares.add(this.lastRouterMiddleware);
      routerMiddlewares.run(req, res);
    } else {
      next();
    }
  }
};

module.exports = Router;
```

라우터를 실행할 때는 미들웨어의 `_run()` 에서 `run()` 을 호출하게 될 것입니다. 라우터의 `run()`은 `request`의 url과 method를 확인합니다. 대응되는 route가 존재하면 관련 미들웨어들을 실행하고, 그렇지 않으면 다음 미들웨어를 실행합니다. 

여기서 핵심은 `req.next = next;` 입니다. 라우터를 실행하다가 모든 작업을 마치면 다시 바깥에 있는 미들웨어로 돌아가야 합니다. 이걸 `request` 객체에 바깥의 `next()`를 저장해 두었다가 라우터의 마지막 미들웨어에서 이를 호출하는 방식으로 구현할 수 있었습니다. `lastRouterMiddleware`는 바깥의 `next()`를 호출하는 함수로 `run()`을 실행할 때 항상 마지막 미들웨어가 되도록 미들웨어 배열의 마지막에 `push`됩니다.

<img width="702" alt="express.js" src="https://user-images.githubusercontent.com/18232901/60753640-d5e53500-a010-11e9-9a2d-b48f8f5c0197.png">

나머지 메소드는 라우터 객체를 초기화하고 각 경로에 대응하는 로직을 추가할 때 사용하게 됩니다.

#### 미들웨어 수정

라우터도 미들웨어에 등록될 수 있어야 하므로 `add()`를 변경할 필요가 있습니다. 바뀐 모습은 이렇습니다.

```javascript
const add = (path, fn) => {
    if(typeof path === 'string'){
      fn.basePath = path;
      const boundRun = fn.run.bind(fn);
      middlewareContainer.push(boundRun);
    } else if (typeof path === 'function'){
      // In this branch, Path is a callback function.
      fn = path;
      middlewareContainer.push(fn);
    } else {
      throw new Error('Add router or middleware');
    }
  };
```

라우터를 등록할 때는 경로와 라우터가 동시에 나오기 때문에 parameter의 개수가 2개가 되었습니다. 첫 번째 argument의 타입이 `string`이면 라우터를 추가하는 것으로 인식합니다. 미들웨어 배열에는 항상 callable 객체가 있어야 하므로 라우터의 `run()` 메소드를 현재 라우터 객체와 바인딩한 함수를 넣게 했습니다.

## 아쉬운 점

미들웨어와 `next()`의 호출이 재귀적이기 때문에 콜 스택에 계속 쌓이는 문제가 있습니다. 이 부분은 어떻게 최적화 할 수 있을지 고민 중입니다. 최근에 든 생각은 제너레이터와 이터레이터, 그리고 tail call optimization을 사용해 보는 건데, 확실하진 않은 것 같네요.

## Reference

미들웨어 구현에 많은 도움을 주신 김정환님의 블로그 - [링크](http://jeonghwan-kim.github.io/)
미들웨어 이미지(wikipedia) - [링크](https://en.wikipedia.org/wiki/Middleware)
Express.js 미들웨어 이미지 - [링크](https://developer.okta.com/blog/2018/09/13/build-and-understand-express-middleware-through-examples)