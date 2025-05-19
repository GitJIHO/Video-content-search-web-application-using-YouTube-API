# YouTube API를 활용한 동영상 컨텐츠 검색 웹 애플리케이션

## 1. 문제 정의

현대 사회에서 YouTube는 가장 큰 동영상 플랫폼으로, 방대한 양의 컨텐츠가 매일 업로드되고 있습니다. 이러한 방대한 자료 속에서 사용자가 원하는 동영상을 효율적으로 검색하고 접근할 수 있는 간편한 인터페이스가 필요합니다. 본 프로젝트는 사용자 친화적인 웹 인터페이스를 통해 YouTube 동영상을 검색하고 결과를 시각적으로 표현하는 웹 애플리케이션을 개발하여 이 문제를 해결하고자 합니다.

## 2. 요구사항 분석

### 2.1 기능적 요구사항

1. **동영상 검색 기능**: 사용자가 키워드를 입력하여 관련된 YouTube 동영상을 검색할 수 있어야 합니다.
2. **검색 결과 표시**: 검색 결과를 썸네일, 제목, 채널명, 설명 등 주요 정보와 함께 카드 형태로 표시해야 합니다.
3. **동영상 바로가기**: 검색 결과에서 원하는 동영상을 선택하여 YouTube로 바로 이동할 수 있어야 합니다.
4. **API 호출 제한 관리**: YouTube API의 일일 할당량을 고려하여 API 호출을 모니터링하고 관리해야 합니다.

### 2.2 비기능적 요구사항

1. **반응형 디자인**: 다양한 화면 크기와 디바이스에서 적절하게 표시되어야 합니다.
2. **사용자 친화적 UI**: 직관적이고 사용하기 쉬운 인터페이스를 제공해야 합니다.
3. **에러 처리**: API 호출 실패 등 예외 상황에 대한 적절한 에러 처리와 사용자 피드백을 제공해야 합니다.
4. **성능 최적화**: 빠른 응답 시간과 효율적인 리소스 사용으로 사용자 경험을 향상시켜야 합니다.

## 3. 기술 스택 및 아키텍처

### 3.1 기술 스택

**백엔드**:
- **Flask**: 파이썬 기반의 경량 웹 프레임워크로, RESTful API 구현에 사용
- **Google API Python Client**: YouTube Data API v3에 접근하기 위한 라이브러리
- **Pyngrok**: 로컬 서버를 인터넷에 노출시키기 위한 ngrok의 Python 래퍼

**프론트엔드**:
- **React**: 사용자 인터페이스 구현을 위한 JavaScript 라이브러리
- **Bootstrap 5**: 반응형 디자인과 UI 컴포넌트를 위한 CSS 프레임워크
- **Babel**: JSX 및 최신 JavaScript 문법을 지원하기 위한 트랜스파일러

### 3.2 시스템 아키텍처

본 프로젝트는 클라이언트-서버 아키텍처를 따르고 있습니다:

1. **클라이언트 (프론트엔드)**:
   - React 기반 단일 페이지 애플리케이션(SPA)
   - 사용자 입력을 받고 검색 결과를 시각적으로 표시하는 UI 컴포넌트
   - RESTful API를 통해 백엔드와 통신

2. **서버 (백엔드)**:
   - Flask 웹 서버
   - YouTube API와의 통신 담당
   - 클라이언트의 요청 처리 및 응답 반환
   - API 호출 횟수 추적 및 관리

3. **외부 서비스**:
   - YouTube Data API v3: 동영상 검색 및 메타데이터 제공
   - Ngrok: 로컬 개발 환경을 인터넷에 노출

4. **데이터 흐름**:
   - 사용자가 검색어 입력 → 프론트엔드에서 백엔드로 API 요청 → 백엔드에서 YouTube API 호출 → 검색 결과를 백엔드에서 프론트엔드로 전달 → 프론트엔드에서 결과 표시

## 4. 핵심 알고리즘 및 처리 메커니즘

### 4.1 YouTube API 연동 메커니즘

```python
def search_with_retry(youtube, query, max_results, retries=3):
    for attempt in range(retries):
        try:
            search_response = youtube.search().list(
                q=query,
                part="snippet",
                maxResults=max_results,
                type="video"
            ).execute()
            return search_response
        except Exception as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise e
```

이 함수는 YouTube API 호출 시 발생할 수 있는 일시적인 오류를 처리하기 위한 재시도 메커니즘을 구현합니다. 주요 특징:

1. **지수 백오프(Exponential Backoff)**: 재시도 간 대기 시간을 점진적으로 늘려 서버 부하를 줄입니다.
2. **최대 재시도 횟수 제한**: 무한정 재시도하지 않고 지정된 횟수만큼만 시도합니다.
3. **예외 처리**: API 호출 실패 시 적절한 예외 처리를 통해 애플리케이션 안정성을 보장합니다.

### 4.2 API 호출 횟수 관리 알고리즘

```python
api_calls = {
    "count": 0,
    "last_reset": time.time()
}

def reset_counter():
    while True:
        time.sleep(86400)  # 24시간(초 단위)
        api_calls["count"] = 0
        api_calls["last_reset"] = time.time()

counter_thread = threading.Thread(target=reset_counter, daemon=True)
counter_thread.start()
```

YouTube API는 일일 할당량이 존재하므로, 이를 관리하기 위한 카운터 시스템을 구현했습니다:

1. **일일 카운터**: API 호출 횟수를 추적하는 카운터
2. **자동 리셋 메커니즘**: 백그라운드 스레드를 사용하여 24시간마다 카운터를 자동으로 리셋
3. **모니터링 엔드포인트**: `/api/quota` 엔드포인트를 통해 현재 API 사용량을 확인 가능

### 4.3 비동기 검색 처리

프론트엔드에서는 React의 `useState`와 `async/await`를 사용하여 비동기 검색 처리를 구현했습니다:

```javascript
const searchVideos = async (e) => {
    e.preventDefault();
    if (!query.trim()) return;
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch(`/api/search?query=${encodeURIComponent(query)}&max_results=12`);
      const data = await response.json();
      
      if (response.ok) {
        setVideos(data.videos);
      } else {
        setError(data.error || '검색 중 오류가 발생했습니다.');
        setVideos([]);
      }
    } catch (err) {
      setError('서버 연결에 실패했습니다. 잠시 후 다시 시도해주세요.');
      setVideos([]);
    } finally {
      setLoading(false);
    }
};
```

이 메커니즘의 주요 특징:

1. **비동기 요청 처리**: `async/await` 구문을 통한 간결한 비동기 코드
2. **로딩 상태 관리**: 요청 중에는 로딩 상태를 표시하여 사용자 경험 향상
3. **에러 처리**: 다양한 예외 상황에 대한 사용자 친화적인 에러 메시지 제공
4. **UI 상태 관리**: React 상태 관리를 통한 UI 업데이트

## 5. 구현 과정 및 주요 코드 설명

### 5.1 백엔드 구현

**Flask 서버 설정**:
```python
app = Flask(__name__, template_folder='./www', static_folder='./www', static_url_path='/')
CORS(app)
```
- `template_folder`와 `static_folder`를 설정하여 프론트엔드 파일을 서빙
- CORS 설정을 통해 크로스 도메인 요청 허용

**YouTube API 검색 엔드포인트**:
```python
@app.route('/api/search', methods=['GET'])
def search_videos():
    query = request.args.get('query', '')
    query = unquote(query)
    max_results = request.args.get('max_results', 10, type=int)
    
    if not query:
        return jsonify({"error": "검색어를 입력해주세요."}), 400
    
    youtube = googleapiclient.discovery.build(
        "youtube", "v3", developerKey=API_KEY
    )
    
    try:
        # API 호출
        search_response = youtube.search().list(
            q=query,
            part="snippet",
            maxResults=max_results,
            type="video"
        ).execute()
        
        # 결과 데이터 가공
        videos = []
        for item in search_response.get("items", []):
            video_data = {
                "id": item["id"]["videoId"],
                "title": item["snippet"]["title"],
                "description": item["snippet"]["description"],
                "thumbnailUrl": item["snippet"]["thumbnails"]["medium"]["url"],
                "channelTitle": item["snippet"]["channelTitle"],
                "publishedAt": item["snippet"]["publishedAt"]
            }
            videos.append(video_data)
        
        return jsonify({"videos": videos})
        
    except Exception as e:
        print(f"YouTube API 오류: {str(e)}")
        return jsonify({"error": f"YouTube API 오류: {str(e)}"}), 500
```
- URL 파라미터로 검색어와 최대 결과 수를 받음
- YouTube API 클라이언트 초기화 및 검색 요청 수행
- API 응답을 프론트엔드에서 사용하기 쉬운 형태로 가공하여 JSON 반환
- 예외 처리를 통한 오류 상황 대응

### 5.2 프론트엔드 구현

**React 컴포넌트 구조**:
1. **App 컴포넌트**: 애플리케이션의 메인 컴포넌트로, 검색 폼과 결과 표시 영역 포함
2. **VideoCard 컴포넌트**: 각 동영상 검색 결과를 카드 형태로 표시하는 컴포넌트

**상태 관리**:
```javascript
const [query, setQuery] = useState('');
const [videos, setVideos] = useState([]);
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);
```
- `query`: 사용자 검색어
- `videos`: 검색 결과 동영상 목록
- `loading`: 로딩 상태 표시
- `error`: 오류 메시지 저장

**UI 렌더링**:
```javascript
return (
  <div className="container my-5">
    {/* 검색 폼 */}
    <div className="row mb-4">
      <div className="col">
        <h1 className="text-center mb-4">YouTube 동영상 검색</h1>
        <form onSubmit={searchVideos}>
          {/* 검색 입력 필드 */}
        </form>
        {/* 에러 메시지 */}
      </div>
    </div>
    
    {/* 로딩 인디케이터 또는 검색 결과 */}
    {loading ? (
      <div className="loading">
        <div className="spinner-border text-primary" role="status">
          <span className="visually-hidden">Loading...</span>
        </div>
      </div>
    ) : (
      <div className="row">
        {videos.length > 0 ? (
          videos.map((video) => (
            <VideoCard key={video.id} video={video} />
          ))
        ) : (
          !loading && !error && query && (
            <p className="text-center">검색 결과가 없습니다.</p>
          )
        )}
      </div>
    )}
  </div>
);
```
- 조건부 렌더링을 통해 로딩 상태, 에러, 검색 결과를 적절히 표시
- 검색 결과가 있을 경우 `VideoCard` 컴포넌트를 맵핑하여 결과 목록 생성

### 5.3 배포 구성

**Ngrok을 통한 터널링**:
```python
# run_server.py
import subprocess
import time
from pyngrok import ngrok

# Flask 서버 시작
server_process = subprocess.Popen(["python", "app.py"])
print("Flask 서버가 시작되었습니다.")

# ngrok 터널 생성
http_tunnel = ngrok.connect(3000)
print(f"ngrok 터널이 생성되었습니다: {http_tunnel.public_url}")

try:
    # 앱이 계속 실행되도록 대기
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    # 종료 시 프로세스 정리
    server_process.terminate()
    ngrok.kill()
```
- 로컬에서 실행 중인 Flask 서버를 인터넷에 노출하기 위한 ngrok 터널 생성
- 서브프로세스를 통한 Flask 서버 실행 및 관리
- 종료 시 프로세스 정리를 위한 예외 처리

## 6. 해결 과정에서 발생한 문제점과 해결 방법

### 6.1 YouTube API 할당량 제한

**문제**: YouTube API는 일일 할당량이 제한되어 있어, 과도한 API 호출 시 서비스 중단 발생

**해결책**:
- API 호출 카운터 구현으로 사용량 모니터링
- 백그라운드 스레드를 통한 자동 카운터 리셋 시스템 구축
- `/api/quota` 엔드포인트를 통한 현재 API 사용량 확인 기능 제공

### 6.2 비동기 요청 처리 및 UI 상태 관리

**문제**: 사용자 검색 시 API 응답을 기다리는 동안 UI가 멈춘 것처럼 보이는 현상

**해결책**:
- React의 상태 관리를 통한 로딩 인디케이터 구현
- `async/await` 패턴을 사용한 비동기 코드 가독성 향상
- 적절한 에러 처리 및 사용자 피드백 제공

### 6.3 특수 문자가 포함된 검색어 처리

**문제**: URL에 특수 문자가 포함된 검색어 전송 시 인코딩 문제 발생

**해결책**:
```javascript
// 프론트엔드: 검색어 인코딩
const response = await fetch(`/api/search?query=${encodeURIComponent(query)}&max_results=12`);
```

```python
# 백엔드: 검색어 디코딩
query = request.args.get('query', '')
query = unquote(query)
```
- 프론트엔드에서 `encodeURIComponent`를 사용한 검색어 인코딩
- 백엔드에서 `unquote` 함수를 통한 검색어 디코딩

## 7. 습득한 기술 및 인사이트

### 7.1 기술적 역량 강화

1. **Python Flask 웹 서버 구축 능력**:
   - RESTful API 엔드포인트 설계 및 구현
   - 외부 API와의 통합 방법 학습
   - 예외 처리 및 오류 응답 패턴 적용

2. **React 프론트엔드 개발 스킬**:
   - 함수형 컴포넌트와 훅(Hooks) 기반 개발
   - 상태 관리와 조건부 렌더링 활용
   - 비동기 데이터 페칭 및 에러 처리

3. **Google API 통합 경험**:
   - API 키 관리 및 보안 실습
   - YouTube Data API 활용 방법 습득
   - API 응답 데이터 가공 및 활용

4. **개발 환경 구성 및 배포 기술**:
   - ngrok을 활용한 로컬 개발 환경 공개
   - 서버 프로세스 관리 및 자동화

### 7.2 얻은 인사이트

1. **예외 처리의 중요성**: 외부 API 호출과 같은 불확실한 작업에서 견고한 예외 처리가 애플리케이션 안정성에 미치는 영향을 이해

2. **사용자 경험 디자인**: 로딩 상태, 오류 메시지 등 적절한 피드백을 통해 사용자 경험을 향상시키는 방법 학습

3. **API 할당량 관리**: 제한된 자원을 효율적으로 사용하기 위한 모니터링 및 관리 메커니즘의 필요성 인식

4. **반응형 웹 디자인**: 다양한 기기에서 일관된 사용자 경험을 제공하기 위한 반응형 디자인의 중요성 이해

## 8. 결론

본 프로젝트를 통해 YouTube API를 활용하여 동영상 검색 기능을 제공하는 웹 애플리케이션을 성공적으로 구현했습니다. Flask 백엔드와 React 프론트엔드를 통합하여 사용자 친화적인 인터페이스를 구축하였으며, API 호출 관리 및 예외 처리를 통해 안정적인 서비스를 제공할 수 있게 되었습니다.

이 과정에서 웹 애플리케이션 개발의 전체 흐름을 경험하고, 프론트엔드와 백엔드의 통합, 외부 API 활용, 배포 과정 등 다양한 웹 개발 영역에 대한 이해도를 높일 수 있었습니다.
