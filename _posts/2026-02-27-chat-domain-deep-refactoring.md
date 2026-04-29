---
title: 채팅 도메인 리팩토링 — WebSocket 핸들러부터 커서 페이징 DTO까지
date: 2026-02-27
tags: [Spring, Java, 리팩토링, WebSocket, STOMP, JPA, QueryDSL, 채팅, 설계]
permalink: /chat-domain-deep-refactoring/
excerpt: "ChatWebSocketController의 try-catch가 @MessageExceptionHandler와 충돌하고 있었고, 네이티브 쿼리는 MAX(sent_at)로 같은 시각 메시지가 겹치는 문제를 포함하고 있었다."
---

## 개요

채팅 도메인은 다른 팀원이 설계한 코드다. [성능 개선](/chat-subquery-redis-listener/)을 마친 뒤 코드 구조를 점검하면서 진행한 리팩토링이다.

```
발견된 구조 문제:

1. WebSocket 예외 처리 이중화 — try-catch + @MessageExceptionHandler가 같은 예외를 2번 잡음
2. 네이티브 쿼리 MAX(sent_at) — 같은 밀리초에 메시지 2개면 중복 반환
4. 불필요한 엔티티 로딩 — existsById면 충분한 곳에서 findById
5. 35줄 덩어리 메서드 — 검증/이미지 파싱/저장/응답 생성을 전부 한 곳에
6. 40줄 커서 페이징 — 커서 분기/slice/nextCursor/DTO 변환이 인라인
```

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

`try-catch`를 제거하고 `@MessageExceptionHandler`에 예외 처리를 위임하는 방식으로 변경:

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

## 문제 2. 네이티브 쿼리 — MAX(sent_at)으로 같은 시각 메시지가 겹침

채팅방 목록의 마지막 메시지를 가져오는 쿼리는 다음과 같았다.

```sql
-- 수정 전 — 같은 밀리초에 메시지 2개가 들어오면 둘 다 반환됨
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

```
문제 1: 같은 시각에 메시지가 겹침
  같은 밀리초에 메시지 2개 → MAX(sent_at)이 같은 값 → JOIN 결과에 2행
  → 채팅방 목록에 마지막 메시지가 2개 표시됨

문제 2: 네이티브 쿼리의 한계
  컴파일 타임 검증 없음 + JOIN FETCH 사용 불가 → N+1 열려있음
```

`MAX(messageId)`로 교체하고 JPQL로 전환:

```java
// 수정 후 — MAX(messageId)로 중복 제거 + JPQL
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

```
변경 효과:
  MAX(sent_at) → MAX(messageId): auto-increment PK라 같은 값이 나올 수 없음
  네이티브 → JPQL: 컴파일 타임 검증 가능
  JOIN FETCH m.user 추가: N+1 제거
  deleted = 0 → m.deleted = false: DB 방언 독립
```

---

## 문제 3. ChatRoomCommandService — 불필요한 엔티티 로딩

`joinScheduleChatRoom`에서 `Schedule` 엔티티를 로딩하고 있었는데, 실제로 사용하는 건 `scheduleId`뿐이었다.

```java
// 수정 전 — Schedule 엔티티 불필요하게 로딩
Schedule schedule = scheduleRepository.findById(scheduleId)
        .orElseThrow(() -> new CustomException(ErrorCode.SCHEDULE_NOT_FOUND));
boolean isMember = userScheduleRepository
        .findByUserAndSchedule(currentUser, schedule).isPresent();  // 엔티티 파라미터
```

```
수정:
  Schedule 엔티티 로딩 → existsByUser_UserIdAndSchedule_ScheduleId()로 교체
  → 의존성 6개 → 5개, SELECT → EXISTS

  검증 순서 재배치:
    수정 전: 멤버십 확인 → 존재 확인
    수정 후: 존재 확인 → 멤버십 확인 → 채팅방 조회
    → 채팅방이 없으면 멤버십 확인 자체가 의미 없으니 먼저 걸러내는 게 맞음

  중복 UserChatRoom.save → saveMember() 헬퍼로 추출
```

---

## 문제 4. ChatRoomQueryService — SELECT → EXISTS

```java
// 수정 전 — 엔티티 조회 후 사용 여부 미확인
Club club = clubRepository.findById(clubId)
        .orElseThrow(() -> new CustomException(ErrorCode.CLUB_NOT_FOUND));
UserClub userClub = userClubRepository.findByUserAndClub(currentUser, club)
        .orElseThrow(() -> new CustomException(ErrorCode.CLUB_NOT_JOIN));
// club, userClub 변수 이후 사용 없음
```

```
수정:
  findById → existsById + existsByUser_UserIdAndClub_ClubId()
    존재 여부만 필요한 곳에 엔티티 전체를 로딩할 필요 없음

  findLastMessages() 헬퍼 추출
    조회 로직을 서비스 메서드에서 분리
```

---

## 문제 5. saveMessage — 35줄 덩어리 메서드

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

이미지 파싱 로직을 `MessageUtils.resolveStoredText()`로 추출하고, 응답 생성은 이미 존재하는 `ChatMessageResponse.from(Message)`를 재활용:

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

`validateAndExtractImageUrl`의 확장자 검증도 `MessageUtils.isValidImageUrlFormat()` + `hasValidImageExtension()`으로 분리.

---

## 문제 6. MessageQueryService — 40줄 → 13줄

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

`ChatRoomMessageResponse.of()`가 `Message` 리스트에서 직접 `nextCursorId`와 `nextCursorAt`을 추출. nextCursor 추출 책임이 서비스에서 DTO 팩토리 메서드로 이동.

---

## 로그 형식 통일

채팅 도메인은 다른 팀원이 작성한 코드라 로그 형식이 영어 브래킷 방식이었다. 피드·알림 도메인과 일관성을 맞추기 위해 한글 서술 형식으로 통일했다.

기존 영어 브래킷 형식을 한글 서술 형식으로 변경했다. `[Async.SaveMessage] final failure`는 `메시지 비동기 저장 최종 실패:`로, `[Chat.Subscriber] message broadcast:`는 `채팅 메시지 브로드캐스트:`로, `[MessageCommand] JSON serialization`은 `메시지 JSON 직렬화 실패:`로, `Redis pub 발행:`은 `채팅 메시지 발행:`으로 각각 통일했다.

debug 레벨이어야 할 started/completed 로그도 모두 제거했다.

---

## 수정 내역 요약

1. `ChatWebSocketController.java` — `try-catch` 제거, `@MessageExceptionHandler` 분리
2. `MessageRepository.java` — `findLastMessagesByChatRoomIds`: 네이티브 → JPQL, `MAX(sent_at)` → `MAX(messageId)`, `JOIN FETCH m.user` 추가
3. `ChatRoomCommandService.java` — `ScheduleRepository` 의존성 제거, `existsByXxx` 교체, `saveMember()` 헬퍼 추출 (129→118줄)
4. `ChatRoomQueryService.java` — `existsById`, `existsByUserIdAndClubId` 교체
5. `MessageCommandService.java` — `saveMessage`: `resolveStoredText()` 위임, `ChatMessageResponse.from()` 재활용 (35→18줄)
6. `MessageUtils.java` — `isValidImageUrlFormat()`, `hasValidImageExtension()` 추출
7. `MessageQueryService.java` — `getChatRoomMessages`: `fetchSlice()` 헬퍼, `ChatRoomMessageResponse.of()` 팩토리 (40→13줄)
8. `ChatRoomMessageResponse.java` — `of(Long, String, List<Message>, boolean)` 팩토리 메서드 추가
9. 전 도메인 로그 — 한글 서술 형식으로 통일, debug 로그 제거

---

## 정리하며

이번 리팩토링에서 반복적으로 발견된 패턴은 "이미 존재하지만 사용되지 않는 기능"이었다. `ChatMessageResponse.from(Message)`가 있는데 9개 필드를 수동으로 조립하고, `existsByXxx`가 있는데 `findById`를 쓰고, `@MessageExceptionHandler`가 있는데 `try-catch`도 따로 두는 식이다.

```
MAX(sent_at) 문제가 발견하기 어려운 이유:
  로컬에서 1명이 테스트하면 같은 밀리초에 메시지가 겹칠 일이 거의 없음
  초당 수십 건이 전송되는 운영 환경에서야 같은 밀리초 충돌이 발생

해결 원칙:
  시간(sent_at) 기준으로 "가장 최근 1건"을 뽑을 때
  같은 시각에 여러 건이 있으면 어느 것이 "최근"인지 구분할 수 없음
  → auto-increment PK(messageId)처럼 절대 겹치지 않는 값을 기준으로 써야 함
```

> **네이티브 쿼리는 편의성을 주지만, 컴파일 타임 검증과 JOIN FETCH 모두 포기해야 한다.**
> 동등하게 표현 가능하면 JPQL을 우선으로 하되, MySQL 전용 함수(LOG, TIMESTAMPDIFF)가 필요한 경우에만 네이티브를 남긴다.

---

## 시리즈 탐색

**◀ 이전 글**
[멀티 도메인 코드 정리 — 표면 정리부터 Semaphore 제거까지](/multi-domain-cleanup-surface-to-semaphore/)

**▶ 다음 글**
[피드 도메인 리팩토링 — 서비스 책임 분리와 네이티브 쿼리 전환](/feed-domain-442-line-service-split/)
