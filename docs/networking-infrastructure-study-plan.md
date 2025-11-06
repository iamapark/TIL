# 🌐 Phase 4: 네트워크 & 인프라 상세 학습 계획서

## 📋 학습 개요

### 목표
- 현대적 네트워크 아키텍처 및 프로토콜 완전 이해
- 클라우드 네이티브 인프라 구축 및 운영 능력 습득
- 프로덕션 레벨 모니터링 및 보안 시스템 구현
- 마이크로서비스 환경에서의 네트워크 최적화 전문성 확보

### 학습 기간
- **총 기간**: 16주 (4개월)
- **주간 학습 시간**: 20-25시간
- **실습 프로젝트**: 5개 (주요 영역별)

### 전제 조건
- Linux 기본 명령어 사용 능력
- Docker & Kubernetes 기본 지식
- 백엔드 개발 경험 (Spring Boot/Node.js 등)
- 클라우드 기본 개념 이해

---

## 📡 Week 1-4: 네트워크 프로그래밍 기초 (4주)

### 학습 목표
- TCP/IP 프로토콜 스택 심층 이해
- HTTP/2, HTTP/3, QUIC 프로토콜 마스터
- 소켓 프로그래밍 및 네트워크 프로그래밍 실무 적용
- OSI 모델과 실제 네트워크 구현의 관계 파악

### 세부 학습 내용

#### 1.1 TCP/IP 프로토콜 기초 (1주)
- **TCP/IP 4계층 모델 완전 이해**
  - Application Layer (HTTP, FTP, DNS, SMTP)
  - Transport Layer (TCP, UDP)
  - Network/Internet Layer (IP, ICMP, ARP)
  - Network Access Layer (Ethernet, WiFi)

- **TCP vs UDP 심층 분석**
  ```bash
  # TCP 연결 상태 확인
  netstat -an | grep LISTEN
  ss -tuln
  
  # UDP 패킷 모니터링
  tcpdump -i any udp port 53
  
  # 네트워크 인터페이스 분석
  ip addr show
  ip route show
  ```

- **패킷 분석 실습**
  - Wireshark 활용 패킷 캡처
  - tcpdump 명령어 마스터
  - 3-way handshake 분석
  - TCP 재전송 및 혼잡 제어 이해

#### 1.2 HTTP 프로토콜 진화 (1주)
- **HTTP/1.1의 한계점**
  - Head-of-line blocking 문제
  - Connection multiplexing 부족
  - 비효율적인 헤더 전송

- **HTTP/2 심화 학습**
  ```nginx
  # NGINX HTTP/2 설정
  server {
      listen 443 ssl http2;
      server_name example.com;
      
      # HTTP/2 Push 설정
      http2_push /css/main.css;
      http2_push /js/main.js;
      
      # 멀티플렉싱 최적화
      http2_max_concurrent_streams 128;
  }
  ```

- **HTTP/3 & QUIC 프로토콜**
  - UDP 기반 전송의 장점
  - 0-RTT 연결 설정
  - Connection Migration 기능
  - 실제 성능 벤치마크 테스트

#### 1.3 소켓 프로그래밍 실습 (1주)
- **Java/Kotlin 소켓 프로그래밍**
  ```kotlin
  // TCP 서버 구현
  class TcpServer(private val port: Int) {
      fun start() {
          ServerSocket(port).use { serverSocket ->
              println("Server listening on port $port")
              
              while (true) {
                  val clientSocket = serverSocket.accept()
                  
                  // Virtual Threads를 활용한 동시성 처리
                  Thread.ofVirtual().start {
                      handleClient(clientSocket)
                  }
              }
          }
      }
      
      private fun handleClient(socket: Socket) {
          socket.use { client ->
              val input = client.getInputStream().bufferedReader()
              val output = client.getOutputStream().bufferedWriter()
              
              val message = input.readLine()
              println("Received: $message")
              
              output.write("Echo: $message\n")
              output.flush()
          }
      }
  }
  ```

- **NIO & 비동기 소켓 처리**
  ```kotlin
  // Spring Boot WebFlux 비동기 처리
  @RestController
  class AsyncNetworkController {
      
      private val webClient = WebClient.builder()
          .clientConnector(ReactorClientHttpConnector(
              HttpClient.create()
                  .protocol(HttpProtocol.HTTP11, HttpProtocol.H2C)
                  .responseTimeout(Duration.ofSeconds(5))
          ))
          .build()
      
      @GetMapping("/proxy/{service}")
      suspend fun proxyRequest(
          @PathVariable service: String,
          @RequestParam params: Map<String, String>
      ): ResponseEntity<String> {
          return try {
              val response = webClient.get()
                  .uri("http://$service.internal") {
                      params.forEach { (key, value) ->
                          it.queryParam(key, value)
                      }
                  }
                  .awaitExchange { it.awaitBody<String>() }
                  
              ResponseEntity.ok(response)
          } catch (ex: Exception) {
              ResponseEntity.status(HttpStatus.GATEWAY_TIMEOUT)
                  .body("Service unavailable: ${ex.message}")
          }
      }
  }
  ```

#### 1.4 네트워크 성능 최적화 (1주)
- **연결 풀링 및 Keep-Alive**
  ```kotlin
  // HTTP 클라이언트 최적화
  @Configuration
  class HttpClientConfig {
      
      @Bean
      fun optimizedWebClient(): WebClient {
          val connectionProvider = ConnectionProvider.builder("optimized")
              .maxConnections(100)
              .maxIdleTime(Duration.ofSeconds(30))
              .maxLifeTime(Duration.ofMinutes(5))
              .pendingAcquireMaxCount(50)
              .evictInBackground(Duration.ofSeconds(120))
              .build()
              
          val httpClient = HttpClient.create(connectionProvider)
              .protocol(HttpProtocol.H2C)
              .compress(true)
              .keepAlive(true)
              
          return WebClient.builder()
              .clientConnector(ReactorClientHttpConnector(httpClient))
              .build()
      }
  }
  ```

- **네트워크 지연시간 측정 및 분석**
  ```bash
  # 네트워크 성능 측정
  ping -c 10 google.com
  traceroute google.com
  mtr --report-cycles 10 google.com
  
  # 대역폭 테스트
  iperf3 -s  # 서버 모드
  iperf3 -c server-ip -t 30  # 클라이언트 모드
  
  # HTTP 성능 테스트
  curl -w "@curl-format.txt" -o /dev/null -s "http://example.com"
  ```

### 실습 프로젝트 1: 고성능 네트워크 프록시 서버
- **목표**: Java 21 Virtual Threads 기반 L7 프록시 구현
- **기능**:
  - HTTP/2 및 HTTP/3 지원
  - 연결 풀링 및 로드 밸런싱
  - 실시간 성능 메트릭 수집
  - WebSocket 프록싱 지원
- **성능 목표**: 10,000 concurrent connections, 평균 지연시간 10ms 이하
- **완성 기한**: 4주차 말

### 평가 기준
- [ ] TCP/IP 프로토콜 스택 이해도 확인 시험 (90점 이상)
- [ ] HTTP/2, HTTP/3 성능 차이 분석 보고서 작성
- [ ] 소켓 프로그래밍으로 채팅 서버 구현
- [ ] 네트워크 프록시 서버 성능 목표 달성

---

## ⚖️ Week 5-8: 로드 밸런싱 & 프록시 시스템 (4주)

### 학습 목표
- NGINX, HAProxy 등 프로덕션 로드 밸런서 완전 마스터
- 다양한 로드 밸런싱 알고리즘 이해 및 적용
- 헬스 체크, 세션 어피니티, Sticky Session 구현
- 리버스 프록시와 API Gateway 패턴 실습

### 세부 학습 내용

#### 5.1 로드 밸런싱 알고리즘 및 전략 (1주)
- **기본 로드 밸런싱 방식**
  - Round Robin: 순차적 요청 분배
  - Weighted Round Robin: 가중치 기반 분배
  - Least Connections: 최소 연결 수 기반
  - IP Hash: 클라이언트 IP 기반 세션 유지

- **고급 로드 밸런싱 전략**
  ```nginx
  # NGINX 고급 로드 밸런싱 설정
  upstream backend_servers {
      # 가중치 기반 라운드 로빈
      server backend1.example.com weight=3 max_fails=3 fail_timeout=30s;
      server backend2.example.com weight=2 max_fails=3 fail_timeout=30s;
      server backend3.example.com weight=1 backup;
      
      # Least connections 방식
      least_conn;
      
      # 세션 persistence
      ip_hash;
      
      # 헬스 체크 설정
      keepalive 16;
  }
  
  server {
      listen 80;
      server_name api.example.com;
      
      location / {
          proxy_pass http://backend_servers;
          
          # 프록시 헤더 설정
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          
          # 타임아웃 설정
          proxy_connect_timeout 5s;
          proxy_send_timeout 10s;
          proxy_read_timeout 10s;
          
          # 버퍼링 최적화
          proxy_buffering on;
          proxy_buffer_size 4k;
          proxy_buffers 8 4k;
      }
  }
  ```

#### 5.2 HAProxy 고급 설정 (1주)
- **HAProxy 설정 및 최적화**
  ```haproxy
  # /etc/haproxy/haproxy.cfg
  global
      daemon
      maxconn 4096
      user haproxy
      group haproxy
      
      # SSL 설정
      ssl-default-bind-ciphers ECDHE+aes128+SHA:ECDHE+aes256+SHA:RSA+aes128+SHA:RSA+aes256+SHA
      ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
      
  defaults
      mode http
      timeout connect 5000ms
      timeout client 50000ms
      timeout server 50000ms
      
      # 로그 설정
      option httplog
      log global
      
  frontend web_frontend
      bind *:80
      bind *:443 ssl crt /etc/ssl/certs/example.com.pem
      
      # HTTP to HTTPS 리다이렉트
      redirect scheme https if !{ ssl_fc }
      
      # ACL을 통한 라우팅
      acl is_api path_beg /api
      acl is_static path_beg /static
      
      use_backend api_servers if is_api
      use_backend static_servers if is_static
      default_backend web_servers
      
  backend api_servers
      balance roundrobin
      option httpchk GET /health
      
      server api1 10.0.1.10:8080 check inter 2000ms rise 2 fall 3
      server api2 10.0.1.11:8080 check inter 2000ms rise 2 fall 3
      server api3 10.0.1.12:8080 check inter 2000ms rise 2 fall 3 backup
      
  backend web_servers
      balance leastconn
      cookie SERVERID insert indirect nocache
      
      server web1 10.0.2.10:8080 check cookie web1
      server web2 10.0.2.11:8080 check cookie web2
  ```

#### 5.3 헬스 체크 및 장애 복구 (1주)
- **정교한 헬스 체크 구현**
  ```kotlin
  // Spring Boot 헬스 체크 엔드포인트
  @RestController
  @RequestMapping("/actuator")
  class CustomHealthController(
      private val dataSource: DataSource,
      private val redisTemplate: RedisTemplate<String, String>
  ) {
      
      @GetMapping("/health")
      fun health(): ResponseEntity<Map<String, Any>> {
          val healthStatus = mutableMapOf<String, Any>()
          var overallStatus = "UP"
          
          // 데이터베이스 헬스 체크
          val dbStatus = checkDatabaseHealth()
          healthStatus["database"] = dbStatus
          if (dbStatus["status"] != "UP") overallStatus = "DOWN"
          
          // Redis 헬스 체크
          val redisStatus = checkRedisHealth()
          healthStatus["redis"] = redisStatus
          if (redisStatus["status"] != "UP") overallStatus = "DOWN"
          
          // 외부 API 헬스 체크
          val externalStatus = checkExternalServices()
          healthStatus["external"] = externalStatus
          if (externalStatus["status"] != "UP") overallStatus = "DEGRADED"
          
          healthStatus["status"] = overallStatus
          healthStatus["timestamp"] = Instant.now()
          
          val httpStatus = when (overallStatus) {
              "UP" -> HttpStatus.OK
              "DEGRADED" -> HttpStatus.OK  // 여전히 서비스 가능
              else -> HttpStatus.SERVICE_UNAVAILABLE
          }
          
          return ResponseEntity.status(httpStatus).body(healthStatus)
      }
      
      private fun checkDatabaseHealth(): Map<String, Any> {
          return try {
              dataSource.connection.use { conn ->
                  conn.createStatement().use { stmt ->
                      stmt.execute("SELECT 1")
                      mapOf(
                          "status" to "UP",
                          "details" to "Database connection successful"
                      )
                  }
              }
          } catch (ex: Exception) {
              mapOf(
                  "status" to "DOWN",
                  "error" to ex.message,
                  "details" to "Database connection failed"
              )
          }
      }
      
      private fun checkRedisHealth(): Map<String, Any> {
          return try {
              redisTemplate.execute { connection ->
                  connection.ping()
              }
              mapOf(
                  "status" to "UP",
                  "details" to "Redis connection successful"
              )
          } catch (ex: Exception) {
              mapOf(
                  "status" to "DOWN",
                  "error" to ex.message,
                  "details" to "Redis connection failed"
              )
          }
      }
  }
  ```

#### 5.4 세션 어피니티 및 Sticky Session (1주)
- **세션 관리 전략**
  ```kotlin
  // 분산 세션 관리
  @Configuration
  @EnableRedisHttpSession
  class SessionConfig {
      
      @Bean
      fun jedisConnectionFactory(): JedisConnectionFactory {
          val config = JedisPoolConfig().apply {
              maxTotal = 50
              maxIdle = 10
              minIdle = 5
              testOnBorrow = true
              testOnReturn = true
              testWhileIdle = true
          }
          
          return JedisConnectionFactory(
              JedisPoolConfig().apply { setConfig(config) }
          ).apply {
              hostName = "redis.internal"
              port = 6379
              database = 0
              usePool = true
          }
      }
      
      @Bean
      fun sessionRepository(
          connectionFactory: RedisConnectionFactory
      ): RedisIndexedSessionRepository {
          return RedisIndexedSessionRepository(connectionFactory).apply {
              setDefaultMaxInactiveInterval(Duration.ofMinutes(30))
              setRedisKeyNamespace("spring:session")
          }
      }
  }
  ```

- **Custom Load Balancer 구현**
  ```kotlin
  // 커스텀 로드 밸런서
  @Component
  class IntelligentLoadBalancer {
      
      data class ServerInfo(
          val url: String,
          val currentConnections: AtomicInteger = AtomicInteger(0),
          val responseTime: AtomicLong = AtomicLong(0),
          val healthStatus: AtomicBoolean = AtomicBoolean(true),
          val weight: Int = 1
      )
      
      private val servers = listOf(
          ServerInfo("http://backend1:8080", weight = 3),
          ServerInfo("http://backend2:8080", weight = 2),
          ServerInfo("http://backend3:8080", weight = 1)
      )
      
      fun selectServer(request: HttpServletRequest): String? {
          val healthyServers = servers.filter { it.healthStatus.get() }
          if (healthyServers.isEmpty()) return null
          
          return when (LoadBalancingStrategy.INTELLIGENT) {
              LoadBalancingStrategy.ROUND_ROBIN -> roundRobinSelection(healthyServers)
              LoadBalancingStrategy.LEAST_CONNECTIONS -> leastConnectionsSelection(healthyServers)
              LoadBalancingStrategy.WEIGHTED_RESPONSE_TIME -> weightedResponseTimeSelection(healthyServers)
              LoadBalancingStrategy.INTELLIGENT -> intelligentSelection(healthyServers, request)
          }
      }
      
      private fun intelligentSelection(servers: List<ServerInfo>, request: HttpServletRequest): String {
          // CPU 사용률, 메모리, 응답 시간을 종합한 지능형 선택
          return servers.minByOrNull { server ->
              val connectionWeight = server.currentConnections.get() * 0.4
              val responseTimeWeight = server.responseTime.get() * 0.3
              val weightPenalty = (10 - server.weight) * 0.3
              
              connectionWeight + responseTimeWeight + weightPenalty
          }?.url ?: servers.first().url
      }
  }
  ```

### 실습 프로젝트 2: 엔터프라이즈 로드 밸런서
- **목표**: NGINX + HAProxy 기반 고가용성 로드 밸런서 구축
- **기능**:
  - 다중 알고리즘 지원 (Round Robin, Least Conn, IP Hash)
  - 실시간 헬스 체크 및 자동 복구
  - SSL 터미네이션 및 인증서 관리
  - 실시간 대시보드 및 메트릭
- **성능 목표**: 99.9% uptime, 5ms 이하 지연시간 증가
- **완성 기한**: 8주차 말

---

## 🌍 Week 9-12: CDN & Edge Computing (4주)

### 학습 목표
- CDN 아키텍처 및 작동 원리 완전 이해
- Edge Computing과 5G 네트워크 통합 기술
- CloudFlare, AWS CloudFront 등 주요 CDN 서비스 활용
- Edge Functions 및 분산 컴퓨팅 구현

### 세부 학습 내용

#### 9.1 CDN 아키텍처 및 원리 (1주)
- **CDN 핵심 개념**
  - Origin Server vs Edge Server
  - Cache Hit/Miss 최적화 전략
  - Geographic Distribution 및 POP (Point of Presence)
  - Cache Invalidation 및 Purge 전략

- **CloudFlare Workers 구현**
  ```javascript
  // Edge Function 예제
  export default {
    async fetch(request, env, ctx) {
      const url = new URL(request.url);
      
      // A/B 테스트 구현
      const isTestGroup = Math.random() < 0.5;
      if (url.pathname === '/api/feature' && isTestGroup) {
        return await handleNewFeature(request);
      }
      
      // 지역별 라우팅
      const country = request.cf.country;
      if (['KR', 'JP', 'CN'].includes(country)) {
        return await forwardToAsiaRegion(request);
      }
      
      // 캐시 전략
      const cacheKey = new Request(url.toString(), request);
      const cache = caches.default;
      let response = await cache.match(cacheKey);
      
      if (!response) {
        response = await fetch(request);
        
        // 캐시 헤더 설정
        const modifiedResponse = new Response(response.body, {
          status: response.status,
          statusText: response.statusText,
          headers: {
            ...response.headers,
            'Cache-Control': 'public, max-age=3600',
            'CDN-Cache-Control': 'max-age=86400'
          }
        });
        
        ctx.waitUntil(cache.put(cacheKey, modifiedResponse.clone()));
        return modifiedResponse;
      }
      
      return response;
    }
  };
  
  async function handleNewFeature(request) {
    // 새로운 기능 처리
    const response = await fetch('https://api-v2.example.com' + new URL(request.url).pathname, {
      method: request.method,
      headers: request.headers,
      body: request.body
    });
    
    return response;
  }
  ```

#### 9.2 Edge Computing 구현 (1주)
- **Edge Functions 개발**
  ```typescript
  // AWS Lambda@Edge 함수
  export const handler: CloudFrontRequestHandler = async (event) => {
    const request = event.Records[0].cf.request;
    const headers = request.headers;
    
    // 모바일 기기 감지 및 최적화
    const userAgent = headers['user-agent']?.[0]?.value || '';
    const isMobile = /Mobile|Android|iPhone|iPad/i.test(userAgent);
    
    if (isMobile && request.uri.includes('/images/')) {
      // 모바일용 이미지 최적화
      request.uri = request.uri.replace('/images/', '/images/mobile/');
      request.querystring += '&format=webp&quality=80';
    }
    
    // 지역별 컨텐츠 개인화
    const country = headers['cloudfront-viewer-country']?.[0]?.value;
    if (country === 'KR') {
      headers['accept-language'] = [{ key: 'Accept-Language', value: 'ko-KR,ko;q=0.9' }];
    }
    
    // 보안 헤더 추가
    headers['x-content-type-options'] = [{ key: 'X-Content-Type-Options', value: 'nosniff' }];
    headers['x-frame-options'] = [{ key: 'X-Frame-Options', value: 'DENY' }];
    headers['x-xss-protection'] = [{ key: 'X-XSS-Protection', value: '1; mode=block' }];
    
    return request;
  };
  ```

#### 9.3 캐시 최적화 전략 (1주)
- **다층 캐시 구조 설계**
  ```kotlin
  // Spring Boot 다층 캐시 구현
  @Service
  class MultiLevelCacheService(
      private val redisTemplate: RedisTemplate<String, String>,
      @Autowired @Qualifier("localCache") private val localCache: Cache
  ) {
      
      @Cacheable(value = ["userProfiles"], key = "#userId")
      fun getUserProfile(userId: String): UserProfile? {
          // L1: Local Cache (Caffeine)
          localCache.get(userId)?.let { return it as UserProfile }
          
          // L2: Redis Cache
          val cached = redisTemplate.opsForValue().get("user:$userId")
          if (cached != null) {
              val profile = objectMapper.readValue(cached, UserProfile::class.java)
              localCache.put(userId, profile)
              return profile
          }
          
          // L3: Database
          val profile = userRepository.findById(userId) ?: return null
          
          // 역방향 캐시 업데이트
          val serialized = objectMapper.writeValueAsString(profile)
          redisTemplate.opsForValue().set("user:$userId", serialized, Duration.ofMinutes(30))
          localCache.put(userId, profile)
          
          return profile
      }
      
      @CacheEvict(value = ["userProfiles"], key = "#userId")
      fun invalidateUserProfile(userId: String) {
          localCache.evict(userId)
          redisTemplate.delete("user:$userId")
          
          // CDN 캐시 무효화 (CloudFlare API 예시)
          invalidateCDNCache("/api/users/$userId")
      }
      
      private fun invalidateCDNCache(path: String) {
          val client = WebClient.builder()
              .defaultHeader("Authorization", "Bearer ${cloudflareBearerToken}")
              .defaultHeader("Content-Type", "application/json")
              .build()
              
          client.post()
              .uri("https://api.cloudflare.com/client/v4/zones/${zoneId}/purge_cache")
              .bodyValue(mapOf("files" to listOf("https://example.com$path")))
              .retrieve()
              .bodyToMono(String::class.java)
              .subscribe()
      }
  }
  ```

#### 9.4 5G 및 Edge AI 통합 (1주)
- **실시간 데이터 처리 Edge 서비스**
  ```kotlin
  // Edge IoT 데이터 처리
  @RestController
  @RequestMapping("/edge")
  class EdgeProcessingController {
      
      @PostMapping("/sensors/batch")
      fun processSensorData(@RequestBody sensorData: List<SensorReading>): EdgeProcessingResult {
          val startTime = System.currentTimeMillis()
          
          // Edge에서 실시간 데이터 전처리
          val processedData = sensorData.parallelStream()
              .filter { it.isValid() }
              .map { reading ->
                  ProcessedSensorData(
                      id = reading.id,
                      timestamp = reading.timestamp,
                      value = reading.value,
                      anomalyScore = calculateAnomalyScore(reading),
                      location = reading.location,
                      processed = true
                  )
              }
              .collect(Collectors.toList())
          
          // 이상 감지된 데이터만 중앙 서버로 전송
          val anomalies = processedData.filter { it.anomalyScore > 0.8 }
          if (anomalies.isNotEmpty()) {
              sendToCloudAsync(anomalies)
          }
          
          // 지역 Edge 스토리지에 저장
          edgeStorageService.saveBatch(processedData)
          
          val processingTime = System.currentTimeMillis() - startTime
          
          return EdgeProcessingResult(
              processedCount = processedData.size,
              anomaliesDetected = anomalies.size,
              processingTimeMs = processingTime,
              edgeLocation = getEdgeLocation()
          )
      }
      
      private fun calculateAnomalyScore(reading: SensorReading): Double {
          // 간단한 이상 탐지 알고리즘 (실제론 ML 모델 사용)
          val historicalAverage = getHistoricalAverage(reading.sensorId)
          val deviation = abs(reading.value - historicalAverage) / historicalAverage
          
          return min(deviation, 1.0)
      }
  }
  ```

### 실습 프로젝트 3: Edge-First 미디어 플랫폼
- **목표**: CDN + Edge Computing 기반 미디어 스트리밍 플랫폼 구축
- **기능**:
  - 지역별 최적화된 컨텐츠 전송
  - 실시간 이미지/비디오 최적화
  - Edge에서 개인화 추천 엔진
  - A/B 테스트 및 트래픽 분할
- **성능 목표**: 전 세계 평균 99ms 이하 지연시간, 99.95% 가용성
- **완성 기한**: 12주차 말

---

## 🔒 Week 13-14: 네트워크 보안 & Zero Trust (2주)

### 학습 목표
- 네트워크 보안 기초부터 고급 기법까지 완전 마스터
- Zero Trust Architecture 설계 및 구현
- TLS/SSL, VPN, 방화벽 등 보안 도구 실무 활용
- 마이크로서비스 환경에서의 보안 Best Practice

### 세부 학습 내용

#### 13.1 TLS/SSL 및 인증서 관리 (0.5주)
- **TLS 1.3 및 최신 암호화 프로토콜**
  ```nginx
  # NGINX TLS 최적화 설정
  server {
      listen 443 ssl http2;
      server_name secure.example.com;
      
      # 인증서 설정
      ssl_certificate /etc/ssl/certs/example.com.crt;
      ssl_certificate_key /etc/ssl/private/example.com.key;
      ssl_trusted_certificate /etc/ssl/certs/ca-chain.crt;
      
      # TLS 1.3 최적화
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
      ssl_prefer_server_ciphers off;
      
      # OCSP Stapling
      ssl_stapling on;
      ssl_stapling_verify on;
      
      # HSTS 설정
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
      
      # 보안 헤더
      add_header X-Frame-Options DENY always;
      add_header X-Content-Type-Options nosniff always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  }
  ```

#### 13.2 Zero Trust Network Architecture (0.5주)
- **Zero Trust 원칙 구현**
  ```kotlin
  // Zero Trust 기반 인증/인가 시스템
  @RestController
  @RequestMapping("/api/secure")
  class ZeroTrustController(
      private val authenticationService: AuthenticationService,
      private val deviceTrustService: DeviceTrustService,
      private val contextualAnalyzer: ContextualAnalyzer
  ) {
      
      @PostMapping("/access")
      fun requestAccess(
          @RequestBody request: AccessRequest,
          httpRequest: HttpServletRequest
      ): ResponseEntity<AccessResponse> {
          
          val context = buildSecurityContext(httpRequest, request)
          
          // 1. Identity Verification
          val identityResult = authenticationService.verifyIdentity(
              request.credentials, context
          )
          if (!identityResult.isValid) {
              return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                  .body(AccessResponse(false, "Identity verification failed"))
          }
          
          // 2. Device Trust Assessment
          val deviceResult = deviceTrustService.assessDevice(
              request.deviceFingerprint, context
          )
          if (deviceResult.riskScore > 0.7) {
              return ResponseEntity.status(HttpStatus.FORBIDDEN)
                  .body(AccessResponse(false, "Device trust score too low"))
          }
          
          // 3. Contextual Analysis
          val contextResult = contextualAnalyzer.analyze(context)
          if (contextResult.hasAnomalies()) {
              // 추가 인증 요구
              return ResponseEntity.ok(
                  AccessResponse(false, "Additional authentication required")
                      .apply { requiresMFA = true }
              )
          }
          
          // 4. Least Privilege Access
          val permissions = calculateMinimalPermissions(
              identityResult.user, request.resource, context
          )
          
          // 5. Continuous Monitoring 시작
          startContinuousMonitoring(identityResult.user, context)
          
          return ResponseEntity.ok(
              AccessResponse(
                  granted = true,
                  token = generateJWT(identityResult.user, permissions, context),
                  permissions = permissions,
                  expiresIn = Duration.ofMinutes(15)
              )
          )
      }
      
      private fun buildSecurityContext(
          httpRequest: HttpServletRequest,
          accessRequest: AccessRequest
      ): SecurityContext {
          return SecurityContext(
              sourceIP = getClientIP(httpRequest),
              userAgent = httpRequest.getHeader("User-Agent"),
              timestamp = Instant.now(),
              geoLocation = geoLocationService.getLocation(getClientIP(httpRequest)),
              networkSegment = determineNetworkSegment(getClientIP(httpRequest)),
              requestedResource = accessRequest.resource,
              deviceInfo = accessRequest.deviceFingerprint
          )
      }
  }
  ```

#### 13.3 마이크로서비스 보안 패턴 (0.5주)
- **Service Mesh 보안**
  ```yaml
  # Istio Security Policy
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: production
  spec:
    mtls:
      mode: STRICT
  ---
  apiVersion: security.istio.io/v1beta1
  kind: AuthorizationPolicy
  metadata:
    name: user-service-authz
    namespace: production
  spec:
    selector:
      matchLabels:
        app: user-service
    rules:
    - from:
      - source:
          principals: ["cluster.local/ns/production/sa/api-gateway"]
      to:
      - operation:
          methods: ["GET", "POST"]
          paths: ["/api/users/*"]
    - from:
      - source:
          principals: ["cluster.local/ns/production/sa/payment-service"]
      to:
      - operation:
          methods: ["GET"]
          paths: ["/api/users/*/profile"]
  ```

#### 13.4 네트워크 침입 탐지 및 대응 (0.5주)
- **실시간 보안 모니터링**
  ```kotlin
  // 네트워크 이상 탐지 시스템
  @Service
  class NetworkAnomalyDetectionService(
      private val metricsCollector: NetworkMetricsCollector,
      private val alertingService: AlertingService
  ) {
      
      @EventListener
      @Async
      fun handleNetworkEvent(event: NetworkEvent) {
          val anomalyScore = calculateAnomalyScore(event)
          
          when {
              anomalyScore > 0.9 -> handleHighRiskEvent(event)
              anomalyScore > 0.7 -> handleMediumRiskEvent(event)
              anomalyScore > 0.5 -> logSuspiciousActivity(event)
          }
      }
      
      private fun calculateAnomalyScore(event: NetworkEvent): Double {
          var score = 0.0
          
          // 비정상적인 요청 패턴 감지
          if (isUnusualRequestPattern(event)) score += 0.3
          
          // 지리적 위치 이상 감지
          if (isUnusualGeolocation(event)) score += 0.2
          
          // 시간대 이상 감지
          if (isUnusualTiming(event)) score += 0.2
          
          // 요청 크기 이상 감지
          if (isUnusualRequestSize(event)) score += 0.2
          
          // 알려진 악성 IP 확인
          if (isKnownMaliciousIP(event.sourceIP)) score += 0.8
          
          return minOf(score, 1.0)
      }
      
      private fun handleHighRiskEvent(event: NetworkEvent) {
          // 즉시 IP 차단
          firewallService.blockIP(event.sourceIP, Duration.ofHours(24))
          
          // 보안팀에 즉시 알림
          alertingService.sendUrgentAlert(
              "High-risk network activity detected",
              "Source IP: ${event.sourceIP}, Activity: ${event.activityType}",
              AlertSeverity.CRITICAL
          )
          
          // 관련 세션 모두 종료
          sessionManager.terminateSessionsFromIP(event.sourceIP)
          
          log.warn("High-risk network event blocked: $event")
      }
  }
  ```

### 실습 프로젝트 4: Zero Trust 마이크로서비스 플랫폼
- **목표**: Zero Trust 원칙 기반 완전한 보안 플랫폼 구축
- **기능**:
  - 마이크로서비스간 mTLS 통신
  - 동적 인증/인가 시스템
  - 실시간 위협 탐지 및 자동 대응
  - 컨텍스트 기반 접근 제어
- **보안 목표**: 0건의 보안 사고, 평균 1초 이내 위협 탐지
- **완성 기한**: 14주차 말

---

## ☸️ Week 15-16: 클라우드 네이티브 & Service Mesh (2주)

### 학습 목표
- Kubernetes 네트워킹 완전 마스터 (CNI, Ingress, NetworkPolicy)
- Istio, Linkerd 등 Service Mesh 실무 활용
- 멀티 클라우드 네트워킹 전략 수립
- Prometheus + Grafana 기반 완전한 모니터링 시스템 구축

### 세부 학습 내용

#### 15.1 Kubernetes 네트워킹 심화 (0.5주)
- **CNI (Container Network Interface) 이해**
  ```yaml
  # Cilium CNI 설정
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: cilium-config
    namespace: kube-system
  data:
    enable-l7-proxy: "true"
    enable-envoy-config: "true"
    enable-hubble: "true"
    enable-hubble-relay: "true"
    enable-metrics: |
      +drop
      +tcp
      +flow
      +icmp
      +http
    hubble-metrics-server: ":9965"
  ---
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: api-network-policy
    namespace: production
  spec:
    podSelector:
      matchLabels:
        app: api-server
    policyTypes:
    - Ingress
    - Egress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            app: nginx-ingress
      ports:
      - protocol: TCP
        port: 8080
    egress:
    - to:
      - podSelector:
          matchLabels:
            app: database
      ports:
      - protocol: TCP
        port: 5432
  ```

#### 15.2 Service Mesh 구현 (1주)
- **Istio 완전 구성**
  ```yaml
  # Istio Gateway 및 VirtualService
  apiVersion: networking.istio.io/v1beta1
  kind: Gateway
  metadata:
    name: api-gateway
    namespace: production
  spec:
    selector:
      istio: ingressgateway
    servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: api-tls-cert
      hosts:
      - api.example.com
  ---
  apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    name: api-routing
    namespace: production
  spec:
    hosts:
    - api.example.com
    gateways:
    - api-gateway
    http:
    - match:
      - uri:
          prefix: "/api/v2/"
      route:
      - destination:
          host: api-service-v2
        weight: 20
      - destination:
          host: api-service-v1
        weight: 80
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 5s
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 2s
  ---
  apiVersion: networking.istio.io/v1beta1
  kind: DestinationRule
  metadata:
    name: api-service-destination
    namespace: production
  spec:
    host: api-service-v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 50
        http:
          http1MaxPendingRequests: 10
          maxRequestsPerConnection: 2
      circuitBreaker:
        consecutiveGatewayErrors: 3
        interval: 30s
        baseEjectionTime: 30s
        maxEjectionPercent: 50
      loadBalancer:
        simple: LEAST_CONN
  ```

#### 15.3 Observability & Monitoring (0.5주)
- **완전한 모니터링 스택**
  ```yaml
  # Prometheus + Grafana + Jaeger Stack
  apiVersion: v1
  kind: Namespace
  metadata:
    name: monitoring
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: Prometheus
  metadata:
    name: prometheus
    namespace: monitoring
  spec:
    serviceAccountName: prometheus
    replicas: 2
    retention: 30d
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    serviceMonitorSelector:
      matchLabels:
        app: api-service
    ruleSelector:
      matchLabels:
        prometheus: main
        role: alert-rules
    alerting:
      alertmanagers:
      - name: alertmanager
        namespace: monitoring
        port: web
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: api-service-monitor
    namespace: monitoring
    labels:
      app: api-service
  spec:
    selector:
      matchLabels:
        app: api-service
    endpoints:
    - port: metrics
      interval: 15s
      path: /actuator/prometheus
      scrapeTimeout: 10s
  ```

#### 15.4 멀티클라우드 네트워킹 (0.5주)
- **클라우드 간 네트워크 연결**
  ```terraform
  # AWS-Azure VPN 연결
  # AWS VPC
  resource "aws_vpc" "main" {
    cidr_block           = "10.0.0.0/16"
    enable_dns_hostnames = true
    enable_dns_support   = true
    
    tags = {
      Name = "main-vpc"
    }
  }
  
  resource "aws_customer_gateway" "azure" {
    bgp_asn    = 65000
    ip_address = azurerm_public_ip.vpn_gateway.ip_address
    type       = "ipsec.1"
    
    tags = {
      Name = "azure-customer-gateway"
    }
  }
  
  resource "aws_vpn_gateway" "main" {
    vpc_id = aws_vpc.main.id
    
    tags = {
      Name = "main-vpn-gateway"
    }
  }
  
  resource "aws_vpn_connection" "azure" {
    customer_gateway_id = aws_customer_gateway.azure.id
    type               = "ipsec.1"
    vpn_gateway_id     = aws_vpn_gateway.main.id
    static_routes_only = false
    
    tags = {
      Name = "aws-azure-vpn"
    }
  }
  ```

### 최종 통합 프로젝트: 글로벌 마이크로서비스 플랫폼
- **목표**: 모든 학습 내용을 통합한 프로덕션 레벨 플랫폼 구축
- **아키텍처**:
  - 멀티 클라우드 (AWS + Azure + GCP)
  - Istio Service Mesh
  - Zero Trust 보안
  - Edge Computing 통합
  - 완전한 Observability
- **성능 목표**:
  - 99.99% 가용성
  - 전 세계 평균 50ms 이하 지연시간
  - 100만 동시 사용자 지원
  - 자동 스케일링 (1초 이내)
- **완성 기한**: 16주차 말

---

## 📊 최종 평가 및 포트폴리오

### 기술 역량 평가
1. **네트워크 기초** (20점)
   - TCP/IP 프로토콜 스택 이해도
   - HTTP/2, HTTP/3, QUIC 실무 활용
   - 소켓 프로그래밍 숙련도
   - 네트워크 성능 최적화 능력

2. **로드밸런싱 & 프록시** (20점)
   - NGINX, HAProxy 설정 및 최적화
   - 고가용성 시스템 구축
   - 헬스체크 및 장애복구 메커니즘
   - 세션 관리 및 스티키 세션

3. **CDN & Edge Computing** (20점)
   - CDN 아키텍처 설계 및 구현
   - Edge Functions 개발
   - 캐시 최적화 전략
   - 실시간 데이터 처리

4. **네트워크 보안** (20점)
   - Zero Trust 아키텍처 구현
   - TLS/SSL 최적화
   - 침입탐지 및 대응 시스템
   - 마이크로서비스 보안

5. **클라우드 네이티브** (20점)
   - Kubernetes 네트워킹 마스터
   - Service Mesh 운영
   - 모니터링 시스템 구축
   - 멀티클라우드 전략

### 포트폴리오 구성
1. **아키텍처 문서**
   - 시스템 설계도
   - 네트워크 토폴로지
   - 보안 정책 문서
   - 운영 매뉴얼

2. **성능 벤치마크 보고서**
   - 부하 테스트 결과
   - 지연시간 분석
   - 처리량 최적화
   - 비용 효율성 분석

3. **실습 프로젝트 코드**
   - GitHub Repository
   - 코드 품질 메트릭
   - 자동화된 테스트
   - CI/CD 파이프라인

---

## 📚 참고 자료 및 리소스

### 필수 도서
- "Computer Networking: A Top-Down Approach" - James Kurose
- "High Performance Browser Networking" - Ilya Grigorik
- "Building Microservices" - Sam Newman
- "Istio in Action" - Christian Posta

### 온라인 강의
- Linux Academy - Kubernetes Networking Deep Dive
- Pluralsight - Advanced Networking on AWS
- Cloud Native Computing Foundation - Service Mesh 101

### 실습 도구
- **네트워크 분석**: Wireshark, tcpdump, ss, netstat
- **부하 테스트**: k6, JMeter, wrk, Apache Bench
- **모니터링**: Prometheus, Grafana, Jaeger, Zipkin
- **시뮬레이션**: GNS3, EVE-NG, Containerlab

### 인증 및 자격증
- AWS Certified Advanced Networking
- Google Cloud Professional Cloud Network Engineer
- Certified Kubernetes Administrator (CKA)
- Istio Certified Associate (ICA)

---

## 🎯 성공을 위한 팁

### 실습 중심 학습
- 매일 최소 3시간 실습 코딩
- 실제 프로덕션 환경 시뮬레이션
- 장애 상황 시나리오 훈련
- 성능 튜닝 경험 축적

### 커뮤니티 참여
- CNCF Slack 참여
- Stack Overflow 질답 활동
- Kubernetes/Istio GitHub 기여
- 기술 컨퍼런스 참가 및 발표

### 지속적 학습
- 최신 네트워킹 트렌드 추적
- 클라우드 서비스 업데이트 모니터링
- 보안 취약점 및 패치 정보 수집
- 성능 최적화 사례 연구

### 실전 경험 쌓기
- 개인 프로젝트에 학습 내용 적용
- 오픈소스 프로젝트 기여
- 사이드 프로젝트로 실전 경험
- 멘토링 및 지식 공유

---

*이 학습 계획서는 네트워크 & 인프라 전문가가 되기 위한 체계적이고 실무 중심의 로드맵입니다. 각 단계를 충실히 이행하여 현대적이고 확장 가능한 네트워크 시스템을 구축할 수 있는 전문성을 확보하시기 바랍니다.*

**최종 업데이트**: 2025년 1월