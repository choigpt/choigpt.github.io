최적화 과정을 이해하려면, 결국 “같은 일을 하더라도 CPU, 메모리, 런타임이 무엇을 더 많이 하게 만드는가”를 보는 감각이 필요합니다. 이번 작업에서 알아야 할 CS 핵심은 아래 정도면 충분합니다.
핵심 개념
•
I/O vs CPU 병목 큰 파일 처리에서는 처음엔 디스크가 느릴 것 같지만, 실제로는 자바에서 바이트마다 메서드 호출, 문자열 생성, 파싱을 하면 CPU가 먼저 병목이 됩니다.
•
객체 생성 비용 String, substring(), Double.parseDouble() 같은 경로는 편하지만 객체 할당과 GC 부담이 큽니다. 1BRC 류 최적화는 “한 줄마다 새 객체를 만들지 않기”가 핵심입니다.
•
파싱 비용 "12.3"을 Double.parseDouble()로 읽는 건 범용적이라 느립니다. 데이터 형식이 고정이면 바이트를 직접 읽어서 123 같은 정수로 만드는 게 훨씬 싸게 먹힙니다.
•
해시 테이블 비용 HashMap<String, Stats>는 편하지만 String 생성, 해시 계산, equals 비교가 누적됩니다. station 키를 바이트 기반으로 다루면 이 비용을 줄일 수 있습니다.
•
메모리 복사 이미 읽어온 바이트를 또 lineBuffer로 한 번 더 복사하면 메모리 대역폭과 CPU를 낭비합니다. 가능한 한 원본 버퍼에서 바로 처리하는 게 좋습니다.
•
캐시 친화성 작은 원시 타입(int, long, byte[]) 위주로 연속 메모리를 다루면 CPU 캐시 효율이 좋아집니다. 반대로 문자열/객체가 많아지면 포인터 추적이 늘어납니다.
•
병렬 처리와 분할 정복 파일을 여러 구간으로 나눠 각 스레드가 독립 처리하면 코어를 활용할 수 있습니다. 다만 세그먼트 시작점은 줄 경계에 맞춰야 하고, 마지막에는 결과 merge 비용이 듭니다.
•
환경 의존성 1BRC 상위권 코드는 Linux, GraalVM native, Unsafe, mmap 전제를 많이 깝니다. Windows + HotSpot JVM에서는 같은 코드가 꼭 더 빠르지 않습니다.
단계별 변경 중간 버전들은 별도 파일로 남기지 않았고, 현재는 최종본만 Main.java에 있습니다. 아래는 이번 대화에서 실제로 거친 단계들을 재구성한 것입니다.
1.
시작점: 바이트 수만 세기
try (InputStream in = new BufferedInputStream(new FileInputStream(filePath))) {
    int b;
    while ((b = in.read()) != -1) {
        totalBytes++;
    }

의미:
•
1바이트씩 읽어서 매우 비효율적
•
1BRC 계산이 아니라 단순 I/O 루프 측정
2.
블록 단위 읽기
byte[] buffer = new byte[1024 * 1024];
while ((n = in.read(buffer)) != -1) {
    totalBytes += n;
}
의미:
•
바이트마다 read() 호출하는 비용 제거
•
디스크/스트림 읽기 기준선 개선
3.
줄 단위 읽기
try (BufferedReader reader = new BufferedReader(new FileReader(filePath), 1024 * 1024)) {
    String line;
    while ((line = reader.readLine()) != null) {
        lineCount++;
        totalChars += line.length();
    }
}
의미:
•
“파일 읽기”에서 “행 처리”로 진입
•
String 생성 시작
4.
station/value 분리
int separator = line.indexOf(';');
String station = line.substring(0, separator);
String value = line.substring(separator + 1);
의미:
•
1BRC 레코드 구조를 실제로 파싱
•
substring() 객체 비용 발생
5.
숫자 파싱 추가
double parsedValue = Double.parseDouble(value);
valueSum += parsedValue;
의미:
•
범용 부동소수 파싱 비용 포함	

String station = line.substring(0, separator);
String value = line.substring(separator + 1);
의미:
•
1BRC 레코드 구조를 실제로 파싱
•
substring() 객체 비용 발생
5.
숫자 파싱 추가
double parsedValue = Double.parseDouble(value);
valueSum += parsedValue;
의미:
•
범용 부동소수 파싱 비용 포함
•
여기서 CPU 사용량이 더 커짐
6.
station별 집계
Map<String, Stats> statsByStation = new HashMap<>();
Stats stats = statsByStation.computeIfAbsent(station, k -> new Stats());
stats.add(parsedValue);
의미:
•
진짜 1BRC 기본형 완성
•
하지만 String 중심이라 아직 느림
7.
readLine() 제거, 바이트 직접 스캔 개념:
•
BufferedInputStream으로 큰 청크를 읽고
•
\n과 ;를 직접 찾음
•
처음엔 줄마다 lineBuffer에 복사해서 처리
의미:
•
BufferedReader 오버헤드 제거
•
대신 구현 복잡도 상승
8.
Double.parseDouble() 제거, 수동 숫자 파싱
private static int parseTemperature(byte[] buf, int start, int end) {
    int sign = 1;
    if (buf[start] == '-') { sign = -1; start++; }
    int value = 0;
    while (start < end) {
        byte c = buf[start++];
        if (c != '.') value = value * 10 + (c - '0');
    }
    return value * sign;

의미:
•
"12.3"을 123으로 바로 파싱
•
큰 성능 개선 포인트
9.
station 문자열 생성 최소화 개념:
•
매 줄 new String(station) 하지 않음
•
조회용 임시 키로 먼저 Map 검색
•
새 station일 때만 바이트 복사 후 저장
의미:
•
station 관련 객체 생성 감소
•
HashMap<String, ...>보다 싸짐
10.
줄 버퍼 복사 제거 개념:
•
청크 안에 완전한 줄은 readBuffer에서 바로 처리
•
청크 끝에 걸린 미완성 줄만 pendingLine에 복사
의미:
•
모든 바이트를 두 번 만지던 구조 제거
•
메모리 복사 비용 절감
11.
파일 청크 병렬 처리 개념:
•
파일을 여러 세그먼트로 나눔
•
세그먼트 시작은 줄 경계로 보정
•
스레드별 partial map 생성
•
마지막에 merge
의미:
•
멀티코어 활용
•
현재 최종 형태
현재 코드 상태 현재 Main.java는 대략 이런 특징을 가집니다.
•
바이트 직접 스캔
•
수동 온도 파싱
•
station 문자열 생성 최소화
•
청크 경계 줄만 복사
•
세그먼트 병렬 처리 + merge
이해 순서 추천 이 과정을 공부용으로 이해하려면 이 순서가 좋습니다.
1.
왜 readLine() + split() + Double.parseDouble()가 느린지
2.
왜 객체 생성이 느린지
3.
왜 고정 형식 데이터는 수동 파싱이 유리한지
4.
왜 HashMap<String, ...>가 비싼지
5.
왜 “복사 제거”가 중요한지
6.
왜 병렬화는 마지막 단계인지

원하면 다음 답변에서 이 단계들을 표로 다시 정리하고, 각 단계마다 “무엇이 병목이었고 무엇을 없앴는지”를 한 줄씩 붙여서 더 공부용으로 정리해드리겠습니다.