---
title: 채팅 도메인 리팩토링 — WebSocket 핸들러부터 커서 페이징 DTO까지
date: 2026-02-27
tags: [Spring, Java, 리팩토링, WebSocket, STOMP, JPA, QueryDSL, 채팅, 설계]
permalink: /chat-domain-deep-refactoring/
excerpt: "ChatWebSocketController의 try-catch가 @MessageExceptionHandler와 충돌하고 있었고, 네이티브 쿼리는 MAX(sent_at) 동시 타이 문제를 안고 있었다. 30줄짜리 메서드가 13줄로, 40줄짜리 커서 페이징이 2줄로 줄어든 과정."
---

## 개요

채팅 도메인도 다른 팀원이 설계한 코드다. 성능 작업을 병행하면서 구조를 파악해나간 부분이라, 원 설계 의도를 완전히 알지 못하는 상태에서 진행한 리팩토링이다.

채팅 도메인의 성능은 이미 정리했다. 이번은 코드 구조 차례였다. `ChatWebSocketController`에서 로그가 두 번 찍히고 있었고, 서비스 메서드는 검증·이미지 파싱·저장·응답 생성을 한꺼번에 처리하는 덩어리 메서드로 남아 있었다. Repository에는 동시 타이 버그를 내포한 네이티브 쿼리가 있었다.

---

## 문제 1. WebSocket 예외 처리 이중화

`ChatWebSocketController.sendMessage()`에 `try-catch`가 있었고, 클래스 레벨에 `@MessageExceptionHandler`도 있었다.

```java
// 수정 전 — try-catch + @MessageExceptionHandler 이중 처리
@MessageMapping("/chat/{roomId}/send")
public void sendMessage(...) {
    try {
        chatService.process(roomId, dto, principal);
    } catch (CustomException e) {
        log.error("[ChatError] ...", e);  // 로그 1
        throw e;  // @MessageExceptionHandler가 또 잡음 → 로그 2
    }
}

@MessageExceptionHandler(CustomException.class)
public void handleCustomException(CustomException e) {
    log.error("[MessageExceptionHandler] ...", e);  // 로그 2 (중복)
    messagingTemplate.convertAndSendToUser(...);
}
```

`try-catch`를 제거했다. `@MessageExceptionHandler`가 도메인 예외 처리와 클라이언트 응답을 담당하고, 예상치 못한 예외는 `Exception.class` catch-all이 처리한다. 비즈니스 로직만 남은 핸들러가 훨씬 읽기 쉬워졌다.

```java
// 수정 후 — 비즈니스 로직만
@MessageMapping("/chat/{roomId}/send")
public void sendMessage(...) {
    chatService.process(roomId, dto, principal);
}

@MessageExceptionHandler(CustomException.class)
public void handleDomainError(CustomException e, SimpMessageHeaderAccessor headerAccessor) { ... }

@MessageExceptionHandler(Exception.class)
public void handleUnexpected(Exception e, SimpMessageHeaderAccessor headerAccessor) { ... }
```

---

## 문제 2. EventListener — 로그만 찍고 다시 throw

`@TransactionalEventListener` 핸들러들이 모두 같은 패턴이었다.

```java
// 124줄짜리 핸들러
@TransactionalEventListener
public void handleChatRoomCreated(ChatRoomCreatedEvent event) {
    log.info("[EventListener] Received ChatRoomCreatedEvent");  // 줄 1
    try {
        // 처리 로직
        log.info("[EventListener] Completed ChatRoomCreatedEvent");  // 줄 2
    } catch (Exception e) {
        log.error("[EventListener] Failed ChatRoomCreatedEvent", e);  // 줄 3
        throw e;  // 잡고 다시 던지기
    }
}
```

로그를 찍고 다시 `throw`하는 건 Spring이 `@TransactionalEventListener` 실패를 이미 로깅하기 때문에 의미가 없다. `try-catch` 전체를 제거했다. 124줄이 73줄로, 77줄짜리 핸들러는 52줄로 줄었다.

`findById` → `getReferenceById`로도 교체했다. 이벤트 컨텍스트에서 이미 존재가 보장된 엔티티는 실제 SELECT 없이 프록시만 가져오는 `getReferenceById`로 충분하다. `IllegalArgumentException` → `IllegalStateException`으로 타입도 수정했다. 이벤트 리스너에서 데이터 불일치는 "잘못된 인자"가 아니라 "시스템 상태 문제"다.

---

## 문제 3. 네이티브 쿼리 — MAX(sent_at) 동시 타이

채팅방 목록의 마지막 메시지를 가져오는 쿼리는 다음과 같이 작성돼 있었다.

```sql
-- 수정 전 — 동시 타이 가능 (같은 시각 메시지가 2개면 둘 다 반환)
SELECT m.* FROM message m
INNER JOIN (
    SELECT chat_room_id, MAX(sent_at) AS max_sent_at
    FROM message
    WHERE chat_room_id IN :chatRoomIds AND deleted = 0
    GROUP BY chat_room_id
) latest ON m.chat_room_id = latest.chat_room_id
         AND m.sent_at = latest.max_sent_at
WHERE m.deleted = 0
```

같은 시각(밀리초 단위)에 메시지 2개가 들어오면 `JOIN` 결과에 2행이 나온다. 채팅방 목록에 마지막 메시지가 2개 표시되거나 중복 처리 문제가 생긴다. 네이티브 쿼리라서 컴파일 타임 검증도 없고, `JOIN FETCH`를 못 써서 N+1도 여전히 열려 있었다.

`MAX(messageId)`로 교체하고 JPQL로 전환했다.

```java
// 수정 후 — MAX(messageId)로 타이 제거 + JPQL
@Query("""
    SELECT m FROM Message m
    JOIN FETCH m.user
    WHERE m.deleted = false
      AND m.chatRoom.chatRoomId IN :chatRoomIds
      AND m.messageId IN (
          SELECT MAX(m2.messageId) FROM Message m2
          WHERE m2.chatRoom.chatRoomId IN :chatRoomIds
            AND m2.deleted = false
          GROUP BY m2.chatRoom.chatRoomId
      )
""")
List<Message> findLastMessagesByChatRoomIds(@Param("chatRoomIds") List<Long> chatRoomIds);
```

`JOIN FETCH m.user`가 추가돼 기존 네이티브 쿼리에 있던 N+1도 함께 제거됐다. `messageId`는 자동 증가 PK라 단조 증가가 보장되므로 타이 걱정이 없다. `deleted = 0` → `m.deleted = false`로 DB 방언 독립 표현으로도 바꿨다.

---

## 문제 4. ChatRoomCommandService — 불필요한 엔티티 로딩

`joinScheduleChatRoom`에서 `Schedule` 엔티티를 로딩하고 있었는데, 실제로 사용하는 건 `scheduleId`뿐이었다.

```java
// 수정 전 — Schedule 엔티티 불필요하게 로딩
Schedule schedule = scheduleRepository.findById(scheduleId)
        .orElseThrow(() -> new CustomException(ErrorCode.SCHEDULE_NOT_FOUND));
boolean isMember = userScheduleRepository
        .findByUserAndSchedule(currentUser, schedule).isPresent();  // 엔티티 파라미터
```

`ScheduleRepository` 의존성을 제거하고 `existsByUser_UserIdAndSchedule_ScheduleId()`로 교체했다. 의존성 6개 → 5개, 엔티티 로딩 → EXISTS 쿼리 1회.

`joinClubChatRoom`의 검증 순서도 재배치했다. 기존엔 멤버십 확인 → 존재 확인 순서였는데, "존재 확인 → 멤버십 확인 → 채팅방 조회" 순으로 바꿨다. Fail-fast 원칙이다. 없는 채팅방에 대해 멤버십 확인을 먼저 하는 건 비효율적이다.

중복된 `UserChatRoom.save` 로직은 `saveMember()` 헬퍼로 추출했다.

---

## 문제 5. ChatRoomQueryService — SELECT → EXISTS

```java
// 수정 전 — 엔티티 조회 후 사용 여부 미확인
Club club = clubRepository.findById(clubId)
        .orElseThrow(() -> new CustomException(ErrorCode.CLUB_NOT_FOUND));
UserClub userClub = userClubRepository.findByUserAndClub(currentUser, club)
        .orElseThrow(() -> new CustomException(ErrorCode.CLUB_NOT_JOIN));
// club, userClub 변수 이후 사용 없음
```

존재 여부만 필요한 곳에 엔티티를 로딩하고 있었다. `existsById` + `existsByUser_UserIdAndClub_ClubId()`로 교체했다.

`BinaryOperator.maxBy`도 제거했다. 채팅방 목록 조회 쿼리가 방당 1건만 반환하도록 JPQL이 이미 보장하고 있는데, 중복 가능성을 가정한 방어 코드가 남아 있었다.

`findLastMessages()` 헬퍼를 추출해 조회 로직을 서비스 메서드에서 분리했다.

---

## 문제 6. saveMessage — 35줄 덩어리 메서드

`MessageCommandService.saveMessage()`가 4가지 일을 한꺼번에 했다.

```java
// 수정 전 — 검증/이미지분기/저장/DTO 9필드 수동 생성
public ChatMessageResponse saveMessage(Long roomId, MessageRequestDto dto, Long userId) {
    ChatRoom chatRoom = chatRoomRepository.findById(roomId)...;
    User user = userRepository.findById(userId)...;
    
    boolean isImage = dto.type() == MessageType.IMAGE;
    String imageUrl = null;
    String storedText = dto.content();
    if (isImage) {
        imageUrl = validateAndExtractImageUrl(dto.content());
        storedText = IMAGE_PREFIX + imageUrl;
    }
    
    Message message = Message.builder()
            .chatRoom(chatRoom).user(user).content(storedText)
            .type(dto.type()).deleted(false).build();
    Message saved = messageRepository.save(message);
    
    return new ChatMessageResponse(
            saved.getMessageId(), saved.getChatRoom().getChatRoomId(),
            saved.getUser().getUserId(), saved.getUser().getNickname(),
            saved.getUser().getProfileImage(), saved.getContent(),
            saved.getType(), saved.isDeleted(), saved.getSentAt());
}
```

이미지 파싱 로직을 `MessageUtils.resolveStoredText()`로 추출하고, 응답 생성은 이미 존재하는 `ChatMessageResponse.from(Message)`를 재활용했다.

```java
// 수정 후 — 18줄
public ChatMessageResponse saveMessage(Long roomId, MessageRequestDto dto, Long userId) {
    ChatRoom chatRoom = chatRoomRepository.findById(roomId)
            .orElseThrow(() -> new CustomException(ErrorCode.CHAT_ROOM_NOT_FOUND));
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new CustomException(ErrorCode.USER_NOT_FOUND));
    
    String storedText = MessageUtils.resolveStoredText(dto);  // 이미지 파싱 위임
    Message saved = messageRepository.save(Message.builder()
            .chatRoom(chatRoom).user(user).content(storedText).type(dto.type()).build());
    
    return ChatMessageResponse.from(saved);  // 기존 팩토리 재활용
}
```

`validateAndExtractImageUrl`의 확장자 검증도 `MessageUtils.isValidImageUrlFormat()` + `hasValidImageExtension()`으로 분리했다.

---

## 문제 7. MessageQueryService — 40줄 → 13줄

커서 페이징 로직이 모두 인라인이었다.

```java
// 수정 전 — 40줄: 커서 분기, slice 자르기, nextCursor 추출, DTO 변환, 6필드 수동 생성
public ChatRoomMessageResponse getChatRoomMessages(Long roomId, Long cursorId, ...) {
    ChatRoom chatRoom = chatRoomRepository.findById(roomId)...;
    String chatRoomName = chatRoom.resolveName();
    
    int fetchSize = pageSize + 1;
    Pageable limit = PageRequest.of(0, fetchSize);
    
    List<Message> slice;
    if (cursorId == null || cursorAt == null) {
        slice = messageRepository.findLatest(roomId, limit);
    } else {
        slice = messageRepository.findOlderThan(roomId, cursorAt, cursorId, limit);
    }
    
    boolean hasMore = slice.size() > pageSize;
    if (hasMore) slice = slice.subList(0, pageSize);
    Collections.reverse(slice);
    
    Long nextCursorId = null;
    LocalDateTime nextCursorAt = null;
    if (!slice.isEmpty()) {
        Message oldest = slice.get(0);
        nextCursorId = oldest.getMessageId();
        nextCursorAt = oldest.getSentAt();
    }
    
    List<ChatMessageResponse> messages = slice.stream()
            .map(ChatMessageResponse::from).toList();
    
    return new ChatRoomMessageResponse(
            roomId, chatRoomName, messages, hasMore, nextCursorId, nextCursorAt);
}
```

`fetchSlice()` 헬퍼와 `ChatRoomMessageResponse.of()` 팩토리 메서드로 분리했다.

```java
// 수정 후 — 13줄
public ChatRoomMessageResponse getChatRoomMessages(Long roomId, Long cursorId, ...) {
    ChatRoom chatRoom = chatRoomRepository.findById(roomId)
            .orElseThrow(() -> new CustomException(ErrorCode.CHAT_ROOM_NOT_FOUND));
    
    int pageSize = clampPageSize(size);
    List<Message> slice = fetchSlice(roomId, pageSize, cursorId, cursorAt);
    boolean hasMore = slice.size() > pageSize;
    if (hasMore) slice = slice.subList(0, pageSize);
    Collections.reverse(slice);
    
    return ChatRoomMessageResponse.of(roomId, chatRoom.resolveName(), slice, hasMore);
}
```

`ChatRoomMessageResponse.of()`가 `Message` 리스트에서 직접 `nextCursorId`와 `nextCursorAt`을 추출한다. nextCursor 추출 책임이 DTO 팩토리 메서드로 이동한 것이다.

---

## 로그 형식 통일

채팅 도메인은 다른 팀원이 작성한 코드라 로그 형식이 영어 브래킷 방식이었다. 피드·알림 도메인과 맞추는 김에 한글 서술 형식으로 통일했다.

| Before | After |
|--------|-------|
| `[Async.SaveMessage] final failure` | `메시지 비동기 저장 최종 실패:` |
| `[Chat.Subscriber] message broadcast:` | `채팅 메시지 브로드캐스트:` |
| `[MessageCommand] JSON serialization` | `메시지 JSON 직렬화 실패:` |
| `Redis pub 발행:` | `채팅 메시지 발행:` |

debug 레벨이어야 할 started/completed 로그도 모두 제거했다.

---

## 수정 내역 요약

| # | 파일 | 변경 내용 |
|---|------|----------|
| 1 | `ChatWebSocketController.java` | `try-catch` 제거, `@MessageExceptionHandler` 분리 |
| 2 | `ChatEventListener.java` | 로그만 찍고 throw하는 try-catch 제거 (124→73줄, 77→52줄) |
| 3 | `ChatEventListener.java` | `findById` → `getReferenceById`, `IllegalArgumentException` → `IllegalStateException` |
| 4 | `MessageRepository.java` | `findLastMessagesByChatRoomIds`: 네이티브 → JPQL, `MAX(sent_at)` → `MAX(messageId)`, `JOIN FETCH m.user` 추가 |
| 5 | `ChatRoomCommandService.java` | `ScheduleRepository` 의존성 제거, `existsByXxx` 교체, `saveMember()` 헬퍼 추출 (129→118줄) |
| 6 | `ChatRoomQueryService.java` | `existsById`, `existsByUserIdAndClubId` 교체, `BinaryOperator.maxBy` 제거 (73→65줄) |
| 7 | `MessageCommandService.java` | `saveMessage`: `resolveStoredText()` 위임, `ChatMessageResponse.from()` 재활용 (35→18줄) |
| 8 | `MessageUtils.java` | `isValidImageUrlFormat()`, `hasValidImageExtension()` 추출 |
| 9 | `MessageQueryService.java` | `getChatRoomMessages`: `fetchSlice()` 헬퍼, `ChatRoomMessageResponse.of()` 팩토리 (40→13줄) |
| 10 | `ChatRoomMessageResponse.java` | `of(Long, String, List<Message>, boolean)` 팩토리 메서드 추가 |
| 11 | 전 도메인 로그 | 한글 서술 형식으로 통일, debug 로그 제거 |

---

## 정리하며

이번 리팩토링에서 가장 자주 발견된 패턴은 "이미 있는데 안 쓰는 것"이었다. `ChatMessageResponse.from(Message)`가 있는데 9개 필드를 수동으로 조립하고, `existsByXxx`가 있는데 `findById`를 쓰고, `@MessageExceptionHandler`가 있는데 `try-catch`도 따로 두는 식이다.

`MAX(sent_at)` 타이 버그는 실제 운영에서 드러나기 전까지 발견하기 어려운 종류다. 초당 수십 건의 메시지가 오가는 인기 채팅방에서, 같은 밀리초에 두 메시지가 도착하는 확률은 낮지 않다. `messageId`는 auto-increment PK라 단조 증가가 보장되므로 타이가 발생할 수 없다. 시간 기반 정렬 기준에는 항상 tie-breaking 컬럼을 함께 써야 한다.

> **네이티브 쿼리는 편의성을 주지만, 컴파일 타임 검증과 JOIN FETCH 모두 포기해야 한다.**
> 동등하게 표현 가능하면 JPQL을 우선으로 하되, MySQL 전용 함수(LOG, TIMESTAMPDIFF)가 필요한 경우에만 네이티브를 남긴다.

---

## 시리즈 탐색

**◀ 이전 글**  
[멀티 도메인 코드 정리 — 표면 정리부터 Semaphore 제거까지](/multi-domain-cleanup-surface-to-semaphore/)

**다음 글 ▶**  
[피드 도메인 리팩토링 — 442줄 서비스를 4개로 쪼갠 기록](/feed-domain-442-line-service-split/)
