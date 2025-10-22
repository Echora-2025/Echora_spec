# Feature Specification: Friends Discovery & Engagement

**Feature Branch**: `004-friends-experiences`  
**Created**: 2025-10-22  
**Status**: Draft  
**Input**: User description: "/friends: GET /friends/map 地理可视化匹配（基于经纬度 + 兴趣标签）, GET /friends/list 相似度推荐（基于标签 / 价值观 / 行为向量）, POST /friends/follow 关注好友, POST /friends/chat 真人聊天, POST /friends/agent_chat agent聊天, DELETE /friends/unfollow 取消关注"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Explore Friends on Map (Priority: P1)

As a user, I want to see nearby people who share my interests on a geo-enabled map so that I can discover meaningful local connections.

**Why this priority**: The map visualization is the signature discovery experience that drives engagement and must work reliably before other flows can succeed.

**Independent Test**: A tester with seeded location and interest data can call `GET /friends/map`, receive clustered pins with profile summaries, and verify privacy rules without invoking follow or chat flows.

**Acceptance Scenarios**:

1. **Given** the requester shares their location and interests, **When** they call the map endpoint, **Then** the response returns geo-clustered candidates filtered by proximity and shared tags, each with safe-to-share preview data.
2. **Given** a user opts out of location sharing, **When** the endpoint is called, **Then** the system either uses coarse location or returns guidance to enable precise matching without exposing hidden users.
3. **Given** there are no nearby matches, **When** the user requests the map, **Then** the system responds with alternate suggestions (e.g., virtual events, broader radius) instead of an empty response.

---

### User Story 2 - Receive Similarity Recommendations (Priority: P1)

As a user, I want a ranked list of people who align with my interests, values, and behavioral signals so that I can decide whom to connect with.

**Why this priority**: Recommendation quality underpins user retention and is critical even without map visuals.

**Independent Test**: A tester can call `GET /friends/list`, inspect the ordered results, and confirm the scoring metadata without needing map or chat features.

**Acceptance Scenarios**:

1. **Given** matching profiles exist, **When** the list endpoint is invoked, **Then** it returns ranked candidates with similarity scores, key shared attributes, and next-step actions.
2. **Given** the user wants to refine matches, **When** they provide filters (e.g., distance, tags, availability), **Then** the system applies them without degrading performance guarantees.
3. **Given** the recommendation engine lacks sufficient signals, **When** the list is requested, **Then** the response includes bootstrapping guidance (e.g., complete profile, add interests).

---

### User Story 3 - Manage Follow Relationships (Priority: P2)

As a user, I want to follow or unfollow people so that my social graph reflects the connections I care about.

**Why this priority**: Social graph management enables ongoing interactions and affects downstream notifications and content.

**Independent Test**: A tester can call `POST /friends/follow` to create a follow relationship, verify idempotency, and then call `DELETE /friends/unfollow` to remove it without using chat features.

**Acceptance Scenarios**:

1. **Given** the requester has permission to follow someone, **When** they submit the follow request, **Then** the system creates or confirms the relationship, updates counters, and returns acknowledgement.
2. **Given** mutual consent rules apply, **When** the target requires approval, **Then** the system places the request in pending state and notifies the target without exposing private data.
3. **Given** the follow relationship already exists or has been removed, **When** follow or unfollow is called again, **Then** the system responds idempotently with current status.

---

### User Story 4 - Real Human Chat (Priority: P2)

As a user, I want to start a direct chat with someone I follow so that we can coordinate in real time.

**Why this priority**: Human-to-human messaging is a key engagement driver once a connection is formed.

**Independent Test**: A tester can call `POST /friends/chat` with a valid conversation payload, see message delivery confirmation, and confirm access controls without invoking agent chat.

**Acceptance Scenarios**:

1. **Given** two users follow each other or otherwise have chat permission, **When** one starts a chat, **Then** the system creates (or resumes) a secure message thread and delivers the message with timestamps.
2. **Given** the sender exceeds rate limits or violates content policies, **When** the chat request is processed, **Then** it is rejected with policy feedback and logged for moderation.

---

### User Story 5 - Agent-Assisted Chat (Priority: P3)

As a user, I want to message an AI agent that represents a friend or community guide so that I can receive quick, contextual assistance when the friend is unavailable.

**Why this priority**: Agent chat complements human chat and provides fallback availability, but is secondary to core social interactions.

**Independent Test**: A tester can call `POST /friends/agent_chat`, confirm the agent responds with context-aware messages, and verify handoff rules independent of human chat success.

**Acceptance Scenarios**:

1. **Given** an agent is enabled for the requested contact, **When** the user sends a message, **Then** the system routes it to the configured agent, returns the agent response, and tags it clearly as automated.
2. **Given** the agent encounters unsupported topics or escalation triggers, **When** the conversation continues, **Then** the system queues a notification for the human friend and communicates expectations to the user.

### Edge Cases

- Location data precision varies across devices; the system must prevent revealing exact home addresses by applying minimum radius or fuzzing rules.
- Users traveling across regions should receive refreshed matches without caching outdated coordinates.
- Following blocked or suspended users must be prevented and produce consistent policy errors.
- Human chat requests to users who disabled messaging require clear rejection messaging without revealing privacy settings.
- Agent chat must degrade gracefully when the AI service is unavailable, offering to notify the human friend instead of failing silently.
- Bulk unfollow actions or automated scripts should respect rate limits and audit logging to avoid abuse.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide `GET /friends/map` that returns geo-clustered friend recommendations including location hints, shared interests, and privacy-safe metadata.
- **FR-002**: System MUST enforce location and interest-based filters, ensuring only consented location sharers appear on the map and respecting visibility settings.
- **FR-003**: System MUST provide `GET /friends/list` delivering ranked candidate profiles with similarity scores, shared attributes, and pagination controls.
- **FR-004**: System MUST allow recommendation filters (radius, interests, availability) and respond within defined latency targets while maintaining relevance.
- **FR-005**: System MUST expose explainability metadata (e.g., top matching factors) for each list entry to build trust.
- **FR-006**: System MUST provide `POST /friends/follow` that creates or confirms follow relationships, respects mutual-consent and block lists, and emits appropriate events.
- **FR-007**: System MUST provide `DELETE /friends/unfollow` that removes follow relationships, updates counters, and returns the resulting relationship state.
- **FR-008**: System MUST provide `POST /friends/chat` that validates mutual permissions, persists messages, triggers delivery notifications, and enforces content and rate policies.
- **FR-009**: System MUST provide `POST /friends/agent_chat` that routes messages to designated agents, returns responses with agent attribution, and logs agent interactions for oversight.
- **FR-010**: System MUST detect when agent chat should hand off to human contacts and surface clear status updates or queue notifications.
- **FR-011**: System MUST maintain activity logs for map views, list recommendations, follow/unfollow actions, and chat interactions to support analytics and compliance.
- **FR-012**: System MUST standardize error responses across endpoints, providing codes, user-facing messages, and remediation hints without revealing sensitive data.

### Key Entities *(include if feature involves data)*

- **FriendRecommendation**: Represents candidate user metadata, similarity scores, explainability factors, and recommendation source (map or list).
- **FollowRelationship**: Captures follower, followee, status (active, pending, blocked), timestamps, and mutual flag.
- **ChatThread**: Stores participants, conversation type (human, agent), last activity, and moderation state.
- **ChatMessage**: Contains sender, recipient(s), content metadata, timestamps, delivery status, and policy review markers.
- **AgentProfile**: Defines agent capabilities, handoff rules, context sources, and escalation contacts tied to users or communities.
- **LocationSnapshot**: Logs user-consented geolocation data with precision level, timestamp, and consent scope for map calculations.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of `GET /friends/map` responses return within 1.5 seconds for regions with under 500 candidates after applying clustering.
- **SC-002**: 95% of `GET /friends/list` requests deliver personalized results within 1 second and reflect at least three shared attributes in explainability metadata.
- **SC-003**: Follow/unfollow actions propagate graph updates to downstream services within 5 seconds for 99% of events.
- **SC-004**: 98% of permitted human chat messages reach the recipient or queue within 2 seconds, and policy enforcement flags <1% false positives during monitoring.
- **SC-005**: Agent chat availability (successful response within 3 seconds) remains above 99% during peak hours, with documented human handoff for the remaining 1%.

### Assumptions

- Location services, recommendation engines, and messaging infrastructure already exist and can be orchestrated through these endpoints.
- Users have provided necessary consent for location sharing, behavioral analysis, and messaging as required by policy.
- Content moderation services are integrated to review chats asynchronously or synchronously based on risk signals.
- Downstream notification systems can handle events for follows, chat messages, and agent handoffs without additional scope in this feature.
