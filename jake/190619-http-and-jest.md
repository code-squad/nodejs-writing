# Jest로 HTTP 서버 테스트 하기

Jest와 Node.js의 http 모듈을 사용해서 간단하게 테스트를 구성하는 방법을 소개합니다.

## 설치

프로젝트가 있는 디렉토리에서 `jest`와 `supertest` 모듈을 받아줍니다.

```console
user@PC ~ $ mkdir my-project && cd my-project
user@PC ~/my-project $ npm init
...
user@PC ~/my-project $ npm install --save-dev jest supertest
```

## HTTP 서버 만들기

서버를 실행하는 파일(`app.js`)에서 `createServer()` 메소드로 서버를 만들고 메소드 체이닝을 통해 `listen()` 메소드로 바로 서버를 실행하도록 코딩하는 것이 일반적입니다. 하지만 테스트를 위해서 서버를 만드는 부분과 실행하는 부분을 두 파일로 나눠 작성합니다.

```javascript
// app.js
const http = require('http');

const server = http.createServer((request, response) => {
  request.on('error', (err) => {
    console.error(err);
    response.statusCode = 400;
    response.end();
  });
  response.on('error', (err) => {
    console.error(err);
  });

  if (request.method === 'GET' && request.url === '/') {
    request.pipe(response);
  } else {
    response.statusCode = 404;
    response.end();
  }
});

module.exports = server;
```

```javascript
// server.js
const server = require('./app');
const port = 8080;

server.listen(port, () => {
  console.log(`Listen in ${port}`);
});
```

## 간단한 테스트 작성

`GET` 메소드로 프로젝트의 루트(`/`)에 접근하는 테스트를 해보겠습니다. `app.js` 파일에서 해당 url에 대한 라우팅이 되어 있으므로response의 상태 코드는 200이어야 합니다. 요청을 보냈을 때 돌아오는 상태 코드가 200인지 테스트해봅시다.

```javascript
//test.js
const request = require('supertest');
const server = require('./app');

describe('Test the root path', () => {
  test('It should response the GET method', (done) => {
    request(server).get('/').then(res => {
      expect(res.statusCode).toBe(200);
      done();
    });
  });
});
```

그리고 `package.json` 파일의 scripts에 아래 내용을 써줍니다.

```json
...
  "scripts": {
    "test": "./node_modules/jest/bin/jest.js"
  },
...
```

## 테스트 실행

콘솔에서 `npm test`를 실행하면 테스트가 실행됩니다.

결과는 아래와 같습니다.

```console
user@PC ~/my-project $  npm test

> my-project@0.0.1 test /Users/user/my-project
> ./node_modules/jest/bin/jest.js

 PASS  ./test.js
  Test the root path
    ✓ It should response the GET method (19ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        1.634s
Ran all test suites.
```