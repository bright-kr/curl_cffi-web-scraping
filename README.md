# Python에서 Web Scraping을 위해 curl\_cffi 사용하기

[![Promo](https://github.com/bright-kr/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/) 

이 가이드는 실제 브라우저의 TLS 핑거프린트를 모방하여 Python에서 Webスクレイピング 스크립트를 강화하기 위해 curl\_cffi를 사용하는 방법을 설명합니다.

- [`curl_cffi`란 무엇입니까?](#what-is-curl_cffi)
- [작동 방식](#how-it-works)
- [Web Scraping을 위해 `curl_cffi`를 사용하는 방법](#how-to-use-curl_cffi-for-web-scraping)
  - [Step #1: 프로젝트 설정](#step-1-project-setup)
  - [Step #2: `curl_cffi` 설치](#step-2-install-curl_cffi)
  - [Step #3: 대상 페이지에 연결](#step-3-connect-to-the-target-page)
  - [Step #4: 데이터 スクレイピング 로직 추가](#step-4-add-the-data-scraping-logic)
  - [Step #5: 모두 합치기](#step-5-put-it-all-together)
- [`curl_cffi`: 고급 사용법](#curl_cffi-advanced-usage)
  - [브라우저 가장(impersonation) 선택](#browser-impersonation-selection)
  - [세션 관리](#session-management)
  - [プロキシ 통합](#proxy-integration)
  - [Async API](#async-api)
  - [WebSockets 연결](#websockets-connection)
- [Web Scraping을 위한 `curl_cffi` vs Requests vs AIOHTTP vs HTTPX](#curl_cffi-vs-requests-vs-aiohttp-vs-httpx-for-web-scraping)
- [Web Scraping을 위한 `curl_cffi` 대안](#curl_cffi-alternatives-for-web-scraping)

## What Is `curl_cffi`?

[`curl_cffi`](https://github.com/lexiforest/curl_cffi)는 CFFI를 통해 `curl-impersonate` 포크에 대한 Python 바인딩을 제공하며, 따라서 브라우저 TLS/JA3/HTTP2 핑거프린트를 가장할 수 있습니다. 이는 [TLS 핑거프린팅](/blog/web-data/tls-fingerprinting)에 기반한 앤チボット 차단을 우회하는 데 도움이 됩니다.

다음은 주요 기능입니다:

- 최신 브라우저 및 커스텀 핑거프린트를 포함한 JA3/TLS 및 HTTP2 핑거프린트 가장 지원
- `requests` 및 `httpx`보다 훨씬 빠르며, `aiohttp`와 유사한 수준의 속도
- `requests` API를 모방함
- 비동기 HTTP リクエスト 수행을 위한 `asyncio` 지원
- 각 リクエスト마다 プロキシ 로테이션 지원
- HTTP/2.0 및 `WebSocket` 지원

## How It Works

HTTPS リクエスト를 전송하면 [TLS 핸드셰이크](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)가 수행되며, 고유한 TLS 핑거프린트를 생성합니다. HTTP 클라이언트는 웹 브라우저와 다르게 동작하기 때문에, 해당 핑거프린트로 자동화를 식별할 수 있으며 이는 앤チボット 방어를 활성화할 수 있습니다.

`curl_cffi`가 기반으로 하는 cURL Impersonate는, 진짜 브라우저의 TLS 핑거프린트를 복제하도록 cURL을 커스터마이즈합니다:

- **TLS 라이브러리 조정**: cURL의 TLS 연결 라이브러리 대신 브라우저가 사용하는 TLS 연결 라이브러리에 의존합니다.
- **설정 변경**: 브라우저를 모방하기 위해 TLS 확장과 SSL 옵션을 조정합니다.
- **HTTP/2 커스터마이징**: 브라우저 핸드셰이크 설정과 일치시킵니다.
- **기본값이 아닌 cURL 플래그**: 정확성을 위해 `--ciphers`, `--curves`, 커스텀 ヘッダー를 설정합니다.

이를 통해 リクエスト가 실제 브라우저에서 온 것처럼 보이게 되어, 봇 탐지를 우회하는 데 도움이 됩니다.

## How to Use `curl_cffi` for Web Scraping

Walmart의 “Keyboard” 페이지를 スクレイピング해 보겠습니다:  

![The Walmart “Keyboard” product page](https://github.com/bright-kr/curl_cffi-web-scraping/blob/main/Images/s_5A13EDF6E0EA32867C0C89DFE864B4C8FA81CE91CC8CE80729F465B232BE7073_1737992236291_image.png)

어떤 HTTP 클라이언트를 사용하든 이 페이지에 접근을 시도하면, 다음과 같은 에러 페이지를 받게 됩니다:  

![Note the response from the server](https://github.com/bright-kr/curl_cffi-web-scraping/blob/main/Images/s_5A13EDF6E0EA32867C0C89DFE864B4C8FA81CE91CC8CE80729F465B232BE7073_1737992185267_image.png)

TLS 핑거프린팅 때문에 실제 브라우저를 시뮬레이션하도록 `User-Agent`를 설정하더라도 이 봇 탐지 페이지가 표시됩니다. 이때 `curl_cffi`가 유용합니다.

### Step #1: Project Setup

머신에 Python 3+가 설치되어 있는지 확인합니다. 그런 다음 `curl_cffi` スクレイピング 프로젝트용 디렉터리를 생성합니다:

```
mkdir curl-cfii-scraper
```

해당 디렉터리로 이동한 후 [가상 환경](https://docs.python.org/3/library/venv.html)을 설정합니다:

```
cd curl-cfii-scraper
python -m venv env
```

선호하는 Python IDE에서 프로젝트 폴더를 열고, 그 폴더에 `scraper.py` 파일을 생성합니다.

IDE의 터미널에서 가상 환경을 활성화합니다. Linux 또는 macOS에서는 다음을 사용합니다:

```bash
./env/bin/activate
```

Windows에서는 다음을 실행합니다:

```bash
env/Scripts/activate
```

### Step #2: Install `curl_cffi`

활성화된 가상 환경에서 HTTP 클라이언트를 설치합니다:

```
pip install curl-cffi
```

### Step #3: Connect to the Target Page

`curl_cffi`에서 `requests`를 임포트합니다:

```python
from curl_cffi import requests
```

이 객체는 Requests와 유사한 고수준 API를 제공합니다. 이를 사용해 대상 페이지에 GET HTTP リクエスト를 수행할 수 있습니다:

```python
response = requests.get("https://www.walmart.com/search?q=keyboard", impersonate="chrome")
```

`impersonate="chrome"` 인자는 `curl_cffi`가 최신 버전의 Chrome에서 온 것처럼 HTTP リクエ스트를 만들도록 지시합니다. 이렇게 하면 Walmart는 자동화된 リクエ스트를 일반 브라우저 リクエ스트로 처리하여 표준 웹 페이지를 반환합니다.

다음과 같이 대상 페이지의 HTML 콘텐츠에 접근할 수 있습니다:

```python
html = response.text
```

`html`을 출력하면 다음이 표시됩니다:

```html
<!DOCTYPE html>
<html lang="en-US">
   <head>
      <meta charSet="utf-8"/>
      <meta property="fb:app_id" content="105223049547814"/>
      <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1, interactive-widget=resizes-content"/>
      <link rel="dns-prefetch" href="https://tap.walmart.com "/>
      <link rel="preload" fetchpriority="high" crossorigin="anonymous" href="https://i5.walmartimages.com/dfw/63fd9f59-a78c/fcfae9b6-2f69-4f89-beed-f0eeb4237946/v1/BogleWeb_subset-Bold.woff2" as="font" type="font/woff2"/>
      <link rel="preload" fetchpriority="high" crossorigin="anonymous" href="https://i5.walmartimages.com/dfw/63fd9f59-a78c/fcfae9b6-2f69-4f89-beed-f0eeb4237946/v1/BogleWeb_subset-Regular.woff2" as="font" type="font/woff2"/>
      <link rel="preconnect" href="https://beacon.walmart.com"/>
      <link rel="preconnect" href="https://b.wal.co"/>
      <title>Electronics - Walmart.com</title>
      <!-- omitted for brevity ... -->
```

### Step #4: Add the Data Scraping Logic

Webスクレイピング을 수행하려면 [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) 같은 HTML 파싱 라이브러리도 필요합니다:

```bash
pip install beautifulsoup4
```

`scraper.py`에서 이를 임포트합니다:

```python
from bs4 import BeautifulSoup
```

이를 사용해 페이지의 HTML을 파싱합니다:

```python
soup = BeautifulSoup(response.text, "html.parser")
```

[`"html.parser"`](https://docs.python.org/3/library/html.parser.html)는 BeautifulSoup가 HTML 문자열을 파싱할 때 사용하는 Python 표준 라이브러리의 기본 HTML 파서입니다. 이 파서에는 페이지의 HTML 요소를 선택하고 그로부터 데이터를 추출하는 메서드들이 포함되어 있습니다.

다음 예시는 페이지 제목만 スクレイピング하는 방법을 보여줍니다. `find()` 메서드를 사용해 [CSS selector](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_selectors)로 선택한 다음 `text` 속성으로 텍스트에 접근할 수 있습니다:

```python
title_element = soup.find("title")
title = title_element.text
```

페이지 제목을 출력합니다:

```python
print(title)
```

### Step #5: Put It All Together

다음은 최종 `curl_cffi` Webスクレイピング 스크립트입니다:

```python
from curl_cffi import requests
from bs4 import BeautifulSoup

# Send a GET request to the Walmart search page for "keyboard"
response = requests.get("https://www.walmart.com/search?q=keyboard", impersonate="chrome")

# Extract the HTML from the page
html = response.text

# Parse the response content with BeautifulSoup
soup = BeautifulSoup(response.text, "html.parser")

# Find the title tag using a CSS selector and print it
title_element = soup.find("title")
# Extract data from it
title = title_element.text

# More complex scraping logic...

# Print the scraped data
print(title)
```

실행합니다:

```bash
python3 scraper.py
```

Windows에서는 다음을 실행합니다:

```bash
python scraper.py
```

결과는 다음과 같습니다:

```
Electronics - Walmart.com
```

`impersonate="chrome"` 인자를 제거하면 대신 다음이 출력됩니다:

```
Robot or human?
```

## `curl_cffi` : Advanced Usage

### Browser Impersonation Selection

`curl_cffi`는 `impersonate` 인자에 전달할 수 있는 고유 라벨을 사용해 여러 브라우저 가장을 지원합니다:

```python
response = requests.get("<YOUR_URL>", impersonate="<BROWSER_LABEL>")
```

다음 라벨을 사용할 수 있습니다:

- `chrome99`, `chrome100`, `chrome101`, `chrome104`, `chrome107`, `chrome110`, `chrome116`, `chrome119`, `chrome120`, `chrome123`, `chrome124`, `chrome131`
- `chrome99_android`, `chrome131_android`
- `edge99`, `edge101`
- `safari15_3`, `safari15_5`, `safari17_0`, `safari17_2_ios`, `safari18_0`, `safari18_0_ios`

다음은 몇 가지 권장 사항입니다:

1.  항상 최신 브라우저 버전을 가장하려면 `chrome`, `safari`, `safari_ios`를 간단히 사용할 수 있습니다.
2.  [Firefox는 현재 사용할 수 없습니다](https://github.com/lexiforest/curl_cffi/issues/59). Firefox는 NSS를 사용하는 반면 다른 브라우저들은 boringssl을 사용하며, curl은 동시에 하나의 TLS 라이브러리에만 링크할 수 있기 때문입니다.
3.  브라우저 버전은 해당 핑거프린트가 변경될 때만 추가됩니다. 어떤 버전이 건너뛰어졌다면, 이전 버전의 ヘッダー를 사용해 그 버전을 가장할 수 있습니다.
4.  브라우저가 아닌 대상의 경우, `ja3`, `akamai` 등의 인자를 사용해 커스텀 TLS 핑거프린트를 지정하십시오. 자세한 내용은 [가장(impersonation) 문서](https://curl-cffi.readthedocs.io/en/latest/impersonate.html)를 참조하십시오.

### Session Management

`curl-cfii`는 [`Session`](https://requests.readthedocs.io/en/latest/user/advanced/#session-objects) 객체를 사용하여 Cookie, ヘッダー 또는 기타 세션별 데이터와 같은 특정 파라미터를 여러 リクエスト에 걸쳐 유지할 수 있습니다.

다음은 코드 예시입니다:

```python
# Create a new session
session = requests.Session()

# This endpoint sets a cookie on the server
session.get("https://httpbin.io/cookies/set/userId/5", impersonate="chrome")

# Print the session's cookies to confirm they are being stored
print(session.cookies)
```

위 스크립트의 출력은 다음과 같습니다:

```
<Cookies[<Cookie userId=5 for httpbin.org />]>
```

이 결과는 세션이 서버에서 정의한 Cookie를 저장하는 등, リクエスト 전반에 걸쳐 상태를 유지하고 있음을 증명합니다.

### Proxy Integration

`requests` 라이브러리와 마찬가지로, `curl_cffi`는 `proxies` 객체를 통해 プロキシ 통합을 지원합니다:

```python
# Define your proxy URL
proxy = "YOUR_PROXY_URL"

# Create a dictionary of proxies for HTTP and HTTPS
proxies = {"http": proxy, "https": proxy}

# Make a request using a proxy and browser impersonation
response = requests.get("<YOUR_URL>", impersonate="chrome", proxies=proxies)
```

### Async API

`curl_cffi`는 [`AsyncSession`](https://curl-cffi.readthedocs.io/en/latest/api.html#curl_cffi.requests.AsyncSession) 객체를 통해 `asyncio` 기반 비동기 リクエスト 수행을 지원합니다:

```python
from curl_cffi.requests import AsyncSession
import asyncio

# Define an async function to execute the asynchronous code
async def fetch_data():
    async with AsyncSession() as session:
        # Perform the asynchronous GET request
        response = await session.get("https://httpbin.org/anything", impersonate="chrome")
        # Print the response text
        print(response.text)

# Run the async function
asyncio.run(fetch_data())
```

`AsyncSession`을 사용하면 여러 비동기 リクエ스트를 효율적으로 처리하기가 더 쉬워집니다.

### WebScokets Connection

`curl_cffi`는 [`WebSocket`](https://curl-cffi.readthedocs.io/en/latest/api.html#curl_cffi.requests.Session.ws_connect) 클래스를 통해 `WebSocket`도 지원합니다:

```python
from curl_cffi.requests import WebSocket


# Define a callback function to handle incoming messages
def on_message(ws, message):
    print(message)

# Initialize the WebSocket connection with the callback
ws = WebSocket(on_message=on_message)

# Connect to a sample WebSocket server and listen for messages
ws.run_forever("wss://api.gemini.com/v1/marketdata/BTCUSD")
```

이는 `WebSocket`을 사용해 데이터를 동적으로 채우는 사이트나 API에서 실시간 데이터를 スクレイピング할 때 특히 유용합니다.

렌더링된 페이지를 スクレイピング하는 대신, 효율적인 데이터 검색을 위해 `WebSocket` 채널을 직접 타겟팅할 수 있습니다.

> **Note**:\
> [`AsyncWebSocket`](https://curl-cffi.readthedocs.io/en/latest/api.html#curl_cffi.requests.AsyncSession.ws_connect) 클래스를 통해 `WebSocket`을 비동기적으로 사용할 수 있습니다.

## curl\_cffi vs Requests vs AIOHTTP vs HTTPX for Web Scraping

다음 표는 Webスクレイピング을 위한 `curl_cffi`와 기타 인기 있는 Python HTTP 클라이언트를 비교합니다:

| **Feature** | **curl\_cffi** | **Requests** | **AIOHTTP** | **HTTPX** |
| --- | --- | --- | --- | --- |
| **Sync API** | ✔️  | ✔️  | ❌   | ✔️  |
| **Async API** | ✔️  | ❌   | ✔️  | ✔️  |
| **Support for** `**WebSocket**`s | ✔️  | ❌   | ✔️  | ❌   |
| **Connection pooling** | ✔️  | ✔️  | ✔️  | ✔️  |
| **Support for HTTP/2** | ✔️  | ❌   | ❌   | ✔️  |
| `**User-Agent**` **customization** | ✔️  | ✔️  | ✔️  | ✔️  |
| **TLS fingerprint spoofing** | ✔️  | ❌   | ❌   | ❌   |
| **Speed** | High | Medium | High | Medium |
| **Retry mechanism** | ❌   | Available via `HTTPAdapter`s | Available only via a third-party library | Available via built-in `Transport`s |
| **Proxy integration** | ✔️  | ✔️  | ✔️  | ✔️  |
| **Cookie handling** | ✔️  | ✔️  | ✔️  | ✔️  |

## `curl_cffi` Alternatives for Web Scraping

`curl_cffi`는 Webスクレイピング에 수동 접근이 필요하며, 대부분의 코드를 직접 작성해야 합니다. 단순한 정적 웹사이트에는 효과적이지만, 동적이거나 보안 수준이 매우 높은 사이트를 다룰 때는 어려울 수 있습니다.

Bright Data는 다음과 같은 `curl_cffi` 대안을 제공합니다:

- [Scraping Browser API](https://brightdata.co.kr/products/scraping-browser): Puppeteer, Selenium, Playwright와 원활하게 통합되는 완전 관리형 클라우드 브라우저 인스턴스입니다. 이 브라우저에는 CAPTCHA 해결과 자동 プロキシ 로테이션이 내장되어 있습니다.
- [Web Scraper APIs](https://brightdata.co.kr/products/web-scraper): 사전 구성된 エンドポイント가 100개 이상의 인기 도메인에서 최신의 구조화된 데이터를 제공합니다.
- [No-Code Scraper](https://brightdata.co.kr/products/web-scraper/no-code): 코딩이 필요 없는 사용자 친화적 온디맨드 데이터 수집 서비스입니다.
- [Datasets](https://brightdata.co.kr/products/datasets): 다양한 웹사이트의 사전 구축 データセット을 탐색하거나, 특정 요구에 맞게 데이터 수집을 맞춤 구성할 수 있습니다.

지금 무료 Bright Data 계정을 생성하여 당사의 プロキシ 및 スクレイピング 솔루션을 테스트해 보십시오!