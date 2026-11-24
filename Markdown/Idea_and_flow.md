Tôi đánh giá đây là **một bước tiến rất hợp lý sau GoldWatch**, và thực tế project này có thể trở thành project portfolio mạnh nhất của bạn nếu triển khai đúng cách.

GoldWatch đã giúp bạn thực hành một hệ thống thiên về data pipeline, crawling, xử lý dữ liệu, EF Core, API, Angular và kiến trúc backend/frontend. Project mới sẽ đưa bạn sang một mảng mà GoldWatch chưa thực sự chạm đến: **transactional systems – hệ thống mà nhiều người cùng thao tác trên một tài nguyên có giới hạn, và correctness quan trọng hơn tốc độ xử lý đơn thuần.**

Đặc biệt, ý tưởng đặt vé sự kiện với seat booking, concurrency, payment và distributed state là một đề tài rất tốt để học backend engineering ở mức sâu hơn.

# **1. Đánh giá tổng thể project**

[Seats booking interactions by Den Klenkov on Dribbble](https://images.openai.com/static-rsc-4/WT6GFZ_4UMdlXdsiSOWIVJrwJ8xnIH1Y1iS9zQOub8hpT_iFIdqlmDy4GRclQ12OAIcYy6Bj77a_V1pVLCssWPRqmJ7jqpdhrQErmb2ynJdH7Uaqol-ZhDJwShynQ4Fr2MnzUtvblGFd5juMNy6vvcYXG5lctyf3Faj1XovgBOc?purpose=inline)

**Event Ticketing Platform**

Transactional Backend + Interactive Seat Booking

ASP.NET Core · PostgreSQL · Redis · Blazor WASM · MVVM

## **Định hướng cốt lõi**

Một nền tảng cho phép:

- Nhà tổ chức tạo event, venue, sơ đồ ghế và mở bán.
- Người dùng xem event, chọn ghế trên interactive seat map.
- Nhiều người có thể cùng tranh chấp một ghế.
- Hệ thống giữ ghế tạm thời trong một khoảng thời gian.
- Người dùng thanh toán và nhận vé.
- Nếu thanh toán thất bại hoặc hết thời gian giữ ghế, ghế được giải phóng.
- Hệ thống phải xử lý payment callback/webhook, retry, duplicate request và lỗi giữa các service.

Điểm đặc biệt của project không phải là CRUD Event hay CRUD User.

> **Giá trị cốt lõi của project là đảm bảo một ghế không thể được bán cho hai người, ngay cả khi hàng trăm request xảy ra đồng thời, mạng bị timeout hoặc payment callback được gửi nhiều lần.**

## **Đánh giá các công nghệ bạn chọn**

| Công nghệ            | Đánh giá                 | Vai trò                                   |
| -------------------- | ------------------------ | ----------------------------------------- |
| ASP.NET Core Web API | Rất phù hợp              | API, business logic, authentication       |
| PostgreSQL           | Rất phù hợp              | Transaction, source of truth, concurrency |
| PostgreSQL locking   | Cực kỳ phù hợp           | Xử lý tranh chấp seat                     |
| Redis                | Phù hợp                  | TTL, cache, coordination, countdown       |
| Outbox Pattern       | Rất đáng học             | Reliable event publishing                 |
| Blazor WebAssembly   | Phù hợp                  | Frontend C#                               |
| MVVM                 | Rất phù hợp để thực hành | State management, UI separation           |
| Payment integration  | Nên có                   | Transaction lifecycle                     |
| Docker               | Nên bắt buộc             | Reproducible environment                  |

Tôi đặc biệt thích việc bạn chọn **Blazor WASM + MVVM**, thay vì tiếp tục Angular.

Đây sẽ là cơ hội để bạn học một architectural pattern ở frontend, đồng thời sử dụng lại C# từ backend sang frontend. Tuy nhiên, có một điều cần lưu ý: Blazor không tự động biến một ứng dụng thành MVVM. Bạn sẽ phải chủ động xây dựng ViewModel, state management, dependency injection, command handling và notification mechanisms.

# **2. Những vấn đề kỹ thuật bạn sẽ thực sự học**

## **2.1. Concurrency – vấn đề lớn nhất của project**

Ví dụ event có một ghế A12.

Trong cùng một thời điểm:

- User A gửi request chọn A12.
- User B gửi request chọn A12.
- Hai request cách nhau chỉ vài milliseconds.
- Cả hai đều nhìn thấy ghế chưa được đặt.

Nếu thiết kế sai:

```
User A: Check seat available → Yes
User B: Check seat available → Yes

User A: Book seat → Success
User B: Book seat → Success
```

Đây chính là race condition.

Bạn sẽ phải tìm hiểu các giải pháp thực tế:

- Optimistic concurrency
- Pessimistic locking
- PostgreSQL `SELECT ... FOR UPDATE`
- Unique constraints
- Isolation levels
- Transaction boundaries
- Deadlocks
- Retry strategies

Đây là kiến thức cực kỳ giá trị khi phỏng vấn Backend Developer.

## **2.2. Seat reservation lifecycle**

Ghế không chỉ có `Available` và `Booked`.

Bạn cần thiết kế state machine.

### **Seat Reservation Lifecycle**

**Available**

**Held / Reserved**

TTL: 5–10 minutes

**Pending Payment**

**Confirmed**

Alternative transitions

Held → Expired → Available

Payment → Failed → Released

Payment → Cancelled

Các transition phải được kiểm soát, không cho phép các thao tác bất hợp lệ như:

- `Confirmed → Available`
- `Expired → Confirmed` mà không có flow hợp lệ.
- User B release reservation của User A.
- Payment callback cũ làm sống lại reservation đã hết hạn.

Đây sẽ là nơi bạn học **domain modeling** thực sự thay vì chỉ tạo Entity và CRUD.

## **2.3. PostgreSQL + Redis: cần phân chia trách nhiệm rõ ràng**

Tôi đề xuất nguyên tắc:

> PostgreSQL quyết định ghế thuộc về ai. Redis hỗ trợ reservation lifecycle và các tác vụ coordination.

Không nên để Redis trở thành nguồn dữ liệu duy nhất quyết định booking thành công.

### **PostgreSQL**

- Event
- Venue
- Seat
- Ticket
- Reservation
- Order
- Payment
- Transaction history
- Unique constraints
- Source of truth

### **Redis**

- Reservation TTL
- Temporary reservation lookup
- Countdown/state hỗ trợ UI
- Rate limiting
- Cache event data
- Idempotency coordination nếu cần

Một thiết kế nguy hiểm là:

```
Redis says seat available
→ User books seat
→ Database insert
```

Bởi Redis và PostgreSQL không có một distributed transaction tự động bảo đảm cả hai cùng thành công.

Do đó, **DB phải luôn có constraint và cơ chế bảo vệ cuối cùng**.

# **3. Kiến trúc tổng thể đề xuất**

Tôi khuyên bạn bắt đầu với **Modular Monolith**, không nên microservices ngay từ đầu.

Bạn vẫn có thể thực hành Outbox, Redis, background workers, message processing và distributed failure mà không cần phải deploy 5–10 service riêng biệt.

**Blazor WebAssembly Client**

Views · ViewModels · UI State · Seat Map

HTTPS · REST API

### **ASP.NET Core Web API**

Events Module

Event catalog

Booking Module

Seat reservation

Payment Module

Payment lifecycle

Ticket Module

Ticket issuance

**PostgreSQL**

Source of truthTransactions · Locks

**Redis**

TTL · CacheTemporary state

**Background Worker**

Outbox processor · Expiration · Notifications

### **Kiến trúc source code**

```
EventBooking/
├── src/
│   ├── EventBooking.Api/
│   │   ├── Controllers/
│   │   ├── Middleware/
│   │   ├── Extensions/
│   │   └── Program.cs
│   │
│   ├── EventBooking.Application/
│   │   ├── Events/
│   │   ├── Bookings/
│   │   ├── Payments/
│   │   ├── Tickets/
│   │   ├── DTOs/
│   │   ├── Interfaces/
│   │   └── Behaviors/
│   │
│   ├── EventBooking.Domain/
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Enums/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   └── Rules/
│   │
│   ├── EventBooking.Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── DbContext/
│   │   │   ├── Configurations/
│   │   │   └── Migrations/
│   │   ├── Redis/
│   │   ├── Payments/
│   │   ├── Outbox/
│   │   └── Services/
│   │
│   ├── EventBooking.Worker/
│   │   ├── OutboxProcessor/
│   │   ├── ReservationExpiration/
│   │   └── BackgroundJobs/
│   │
│   └── EventBooking.Blazor/
│       ├── Pages/
│       ├── Components/
│       ├── ViewModels/
│       ├── Services/
│       ├── State/
│       └── Models/
│
└── tests/
    ├── EventBooking.UnitTests/
    ├── EventBooking.IntegrationTests/
    └── EventBooking.ConcurrencyTests/
```

Không nhất thiết phải tạo hết project này ngay V0. Cấu trúc sẽ được mở rộng dần theo từng version.

# **4. Thiết kế database sơ bộ**

Đây là domain model tôi nghĩ nên hướng đến.

### **Core Entities**

**Event**

Id · Name · Description · StartTime · VenueId · Status

**Venue**

Id · Name · Location · Capacity

**Seat**

Id · VenueId · Section · Row · Number · SeatType

**EventSeat**

EventId · SeatId · Price · Status

Một ghế trong venue có trạng thái và giá riêng cho mỗi event.

**Reservation**

Id · UserId · EventId · Status · ExpiresAt · CreatedAt

**ReservationItem**

ReservationId · EventSeatId · PriceAtReservation

**Order**

Id · UserId · ReservationId · TotalAmount · Status

**Payment**

Id · OrderId · Provider · Amount · Status · ProviderTransactionId

**Ticket**

Id · OrderId · EventSeatId · TicketCode · Status

**OutboxMessage**

Id · Type · Payload · CreatedAt · ProcessedAt · RetryCount

## **Quan trọng: EventSeat**

Tôi không khuyến khích lưu trạng thái availability trực tiếp trên `Seat`.

Ví dụ:

```
Venue
 └── Seat A12
      └── Exists permanently
```

Nhưng:

```
Event #1
 └── A12 → $50

Event #2
 └── A12 → $80

Event #3
 └── A12 → Unavailable
```

Vì vậy nên có:

```
Seat
  ↓
EventSeat
  ├── EventId
  ├── SeatId
  ├── Price
  ├── Status
  └── ReservationId (nullable, nếu thiết kế cần)
```

Điều này giúp bạn có thể tái sử dụng một venue cho nhiều event khác nhau mà không bị lẫn trạng thái ghế.

# **5. Kế hoạch phát triển theo Version**

Tôi sẽ chia roadmap theo kiểu SchoolCRUD/GoldWatch: mỗi version có mục tiêu, kiến thức cần học và tiêu chí hoàn thành.

Dự kiến project khá lớn, khoảng **12–16 tuần nếu học và phát triển đều**, nhưng không cần cố ép deadline. Bạn có thể kéo dài mỗi phase tùy thời gian.

## **Phase 0 — Project Foundation**

### **V0.1 — Project setup**

**Mục tiêu:** Xây dựng nền móng solution.

- Tạo Git repository.
- Setup .NET solution.
- Setup ASP.NET Core Web API.
- Setup PostgreSQL bằng Docker Compose.
- Setup Redis bằng Docker Compose.
- Setup Blazor WebAssembly.
- Environment configuration.
- Git branching / commit conventions.
- Swagger.

### **Docker Compose ban đầu**

```
services:
  postgres:
    image: postgres
    container_name: eventbooking-postgres
    environment:
      POSTGRES_DB: eventbooking
      POSTGRES_USER: app
      POSTGRES_PASSWORD: development_password
    ports:
      - "5432:5432"

  redis:
    image: redis
    container_name: eventbooking-redis
    ports:
      - "6379:6379"
```

Sau đó bổ sung:

- Persistent volume.
- Health checks.
- Environment variables.
- Docker networking.
- Không lưu secret vào Git.

### **Acceptance Criteria**

- API chạy được.
- Blazor WASM chạy được.
- PostgreSQL kết nối được.
- Redis kết nối được.
- API có Swagger.
- Có thể chạy môi trường local bằng một lệnh Docker Compose.

## **V0.2 — Architecture & Domain Modeling**

**Mục tiêu:** Thiết kế domain trước khi bắt đầu CRUD.

Bạn sẽ học:

- Domain-driven design basics.
- Entity.
- Value Object.
- Aggregate.
- Domain invariants.
- Domain events.
- Repository pattern: khi nào cần, khi nào không.
- Application service.
- Dependency inversion.

### **Domain đầu tiên**

```
Venue
Event
Seat
EventSeat
User
```

Chưa cần payment, chưa cần reservation.

### **Bài tập**

Tự thiết kế:

1. Venue có nhiều Seat.
2. Event thuộc Venue.
3. Event có EventSeats.
4. EventSeat có price và status.
5. Event chỉ có thể mở bán khi đủ cấu hình.

### **Acceptance Criteria**

Bạn có thể giải thích:

- Tại sao `Seat` và `EventSeat` là hai entity?
- Entity nào chịu trách nhiệm quản lý state?
- Vì sao không cho Controller trực tiếp sửa database?
- Domain invariant là gì?
- Khi nào một business rule thuộc Domain, Application hay Infrastructure?

# **Phase 1 — Event Management System**

## **V1.0 — Event & Venue CRUD**

Bắt đầu với phần tương đối đơn giản.

### **Backend**

- Venue CRUD.
- Event CRUD.
- Seat CRUD.
- EventSeat generation.
- EF Core.
- PostgreSQL.
- Fluent API.
- Migrations.
- DTO.
- Validation.
- Pagination.
- Filtering.

### **API ví dụ**

```
GET    /api/events
GET    /api/events/{id}
POST   /api/events
PUT    /api/events/{id}
DELETE /api/events/{id}

GET    /api/venues
POST   /api/venues

GET    /api/events/{id}/seats
```

### **Kiến thức**

- EF Core relationship mapping.
- One-to-many.
- Many-to-many.
- Composite key.
- Index.
- Database constraints.
- DTO projection.
- Validation.

### **Acceptance Criteria**

- Tạo event.
- Tạo venue.
- Generate seat layout.
- Trả về seat map của event.
- Không thể tạo seat duplicate.
- Không thể tạo event với venue không tồn tại.

## **V1.1 — Authentication & Authorization**

Tích hợp:

- ASP.NET Core Identity.
- JWT Authentication.
- Refresh token.
- Role-based authorization.

Roles:

```
Admin
Organizer
Customer
```

Ví dụ:

```
Organizer
 ├── Create Event
 ├── Configure Seat Map
 └── Manage Event

Customer
 ├── View Events
 ├── Reserve Seat
 ├── Pay
 └── View Tickets

Admin
 └── Manage Platform
```

### **Acceptance Criteria**

- Register.
- Login.
- Refresh token.
- Logout/revoke token.
- Protected endpoints.
- Organizer không thể chỉnh sửa event của organizer khác.
- Customer không thể truy cập admin endpoints.

# **Phase 2 — Seat Booking Core**

Đây sẽ là phase quan trọng nhất của project.

## **V2.0 — Seat Availability**

**Chưa cho booking. Chỉ xây dựng trạng thái ghế và API đọc.**

### **Status đề xuất**

```
public enum EventSeatStatus
{
    Available,
    Reserved,
    Sold,
    Disabled
}
```

Reservation status:

```
public enum ReservationStatus
{
    Active,
    Expired,
    Completed,
    Cancelled
}
```

Payment status:

```
public enum PaymentStatus
{
    Pending,
    Processing,
    Succeeded,
    Failed,
    Cancelled,
    Refunded
}
```

Lưu ý: Không nên tạo một enum khổng lồ gộp trạng thái của seat, reservation, payment và order.

### **API**

```
GET /api/events/{eventId}/seat-map
GET /api/events/{eventId}/availability
```

### **Frontend**

Blazor:

- Seat map rendering.
- Row/section layout.
- Seat color.
- Legend.
- Selected seat.
- Sold seat.
- Disabled seat.

Ở phase này, UI chỉ cần chọn ghế local, chưa cần reservation server-side.

## **V2.1 — Reservation System**

Đây là bước bắt đầu transactional logic.

### **API**

```
POST /api/reservations
GET  /api/reservations/{id}
POST /api/reservations/{id}/cancel
```

Request:

```
{
  "eventId": "event-id",
  "seatIds": ["A1", "A2"]
}
```

Response:

```
{
  "reservationId": "reservation-id",
  "status": "Active",
  "expiresAt": "2026-11-23T16:10:00Z",
  "seats": ["A1", "A2"]
}
```

### **Reservation rules**

- User không thể reserve ghế đã Sold.
- User không thể reserve ghế đã được người khác giữ.
- User không thể reserve Disabled seat.
- Một reservation có thể chứa nhiều seat.
- Nếu một trong các ghế không reserve được, cả reservation phải rollback.
- Reservation phải có expiration time.
- User chỉ được cancel reservation của chính mình.

### **Transactional behavior**

```
BEGIN TRANSACTION

Lock requested EventSeat rows

Validate all seats

Create Reservation

Update EventSeats

COMMIT
```

Nếu bất kỳ bước nào thất bại:

```
ROLLBACK
```

Đây là version đầu tiên bạn sẽ bắt đầu học transaction thực sự.

## **V2.2 — PostgreSQL Concurrency & Locking Lab**

Tôi đề xuất tách hẳn một version để thực nghiệm, không chỉ viết code booking rồi hy vọng nó hoạt động.

### **Thử nghiệm 1 — Race condition không có locking**

Tạo một test harness:

```
100 concurrent requests
             ↓
       Try booking A12
             ↓
        PostgreSQL
```

Kiểm tra:

```
Successful reservations: ?
Failed reservations: ?
Duplicate booking: ?
```

Bạn cố tình implement một phiên bản sai:

```
var seat = await db.EventSeats
    .FirstOrDefaultAsync(x => x.Id == seatId);

if (seat.Status == EventSeatStatus.Available)
{
    seat.Status = EventSeatStatus.Reserved;
    await db.SaveChangesAsync();
}
```

Sau đó chạy concurrent tests để quan sát race condition.

### **Thử nghiệm 2 — Pessimistic locking**

Tìm hiểu và sử dụng PostgreSQL:

```
SELECT *
FROM event_seats
WHERE id = @seatId
FOR UPDATE;
```

Lock row cho đến khi transaction kết thúc.

### **Thử nghiệm 3 — Multiple seats**

Nếu User A reserve:

```
A1, A2, A3
```

và User B reserve:

```
A3, A4, A5
```

Bạn phải xử lý trường hợp hai transaction lock các seat theo thứ tự khác nhau và gây deadlock.

Một giải pháp cần thực nghiệm là **lock theo thứ tự cố định**:

```
Sort Seat IDs
      ↓
Acquire locks in order
      ↓
Validate
      ↓
Update
```

### **Thử nghiệm 4 — Database constraints**

Ví dụ, đảm bảo không tạo duplicate seat cho cùng một event:

```
CREATE UNIQUE INDEX ux_eventseat_event_seat
ON event_seats(event_id, seat_id);
```

Bạn sẽ học cách phân biệt:

- Application validation.
- Database validation.
- Concurrency protection.
- Transaction atomicity.

### **Acceptance Criteria**

- 100 request tranh 1 ghế → chỉ tối đa 1 reservation thành công.
- 100 request tranh 10 ghế → không có ghế bị bán duplicate.
- Reservation nhiều ghế phải atomic.
- Có thể reproduce race condition bằng test.
- Có test chứng minh cơ chế locking hoạt động.
- Có thể giải thích `FOR UPDATE`, transaction isolation và deadlock.

Đây sẽ là một trong những phần đẹp nhất khi trình bày project trong CV hoặc phỏng vấn.

# **Phase 3 — Redis & Reservation Expiration**

## **V3.0 — Redis Reservation TTL**

Sau khi PostgreSQL đã bảo đảm booking correctness, mới thêm Redis.

Ví dụ:

```
Reservation
    ↓
ExpiresAt = 16:10
    ↓
Redis TTL
    ↓
Countdown in UI
```

Redis key có thể thiết kế như:

```
reservation:{reservationId}
```

Value:

```
{
  "userId": "user-123",
  "eventId": "event-456",
  "seatIds": ["A1", "A2"],
  "expiresAt": "2026-11-23T16:10:00Z"
}
```

TTL:

```
600 seconds
```

### **Những kiến thức cần học**

- Redis key design.
- TTL.
- Redis data types.
- Atomic commands.
- Redis connection management.
- Cache invalidation.
- Redis failure scenarios.

### **Điểm quan trọng**

TTL expiration **không có nghĩa Redis sẽ gọi một callback đúng vào thời điểm key hết hạn**.

Bạn cần thiết kế expiration workflow thực sự.

Ví dụ:

```
Reservation expires
         ↓
Worker scans expired reservations
         ↓
PostgreSQL transaction
         ↓
Verify reservation still active
         ↓
Release seat
         ↓
Mark reservation expired
```

Redis expiration notifications có thể được nghiên cứu thêm, nhưng không nên phụ thuộc duy nhất vào chúng để bảo đảm correctness.

## **V3.1 — Reservation Expiration Worker**

Tạo `EventBooking.Worker`.

Worker có thể chạy định kỳ:

```
Every 5 seconds
      ↓
Find expired reservations
      ↓
Process batches
      ↓
Release seats
      ↓
Update reservation status
```

Ví dụ pseudo-code:

```
while (!stoppingToken.IsCancellationRequested)
{
    await ProcessExpiredReservations();

    await Task.Delay(
        TimeSpan.FromSeconds(5),
        stoppingToken
    );
}
```

Sau này nâng cấp:

- Batch processing.
- Retry.
- Distributed worker coordination.
- Idempotent processing.
- Observability.

### **Acceptance Criteria**

- Reservation có TTL.
- Countdown frontend đồng bộ theo server time.
- Hết hạn tự release seat.
- Worker restart không làm reservation bị stuck.
- Worker chạy hai instance không gây lỗi double processing.
- Redis restart không làm database mất correctness.

# **Phase 4 — Order & Payment**

Đây là nơi transactional complexity tăng mạnh.

## **V4.0 — Order Lifecycle**

Không nên nhảy ngay vào tích hợp payment provider.

Đầu tiên tạo Order domain.

```
Reservation
     ↓
Create Order
     ↓
Order Pending
     ↓
Payment Pending
```

Order có thể có:

```
public enum OrderStatus
{
    Pending,
    Paid,
    Cancelled,
    Expired,
    Refunded
}
```

Lưu lại giá tại thời điểm mua:

```
EventSeat current price: 500,000

Reservation created:
PriceAtReservation: 500,000

Event price changes to:
600,000

Existing order:
Still 500,000
```

Đây là một business rule rất quan trọng.

### **Acceptance Criteria**

- Reservation tạo Order.
- Order snapshot giá tại thời điểm đặt.
- Order có total amount.
- Order không bị tạo duplicate.
- Order có lifecycle hợp lệ.

## **V4.1 — Payment Integration**

Bạn có thể tích hợp một payment provider phù hợp môi trường Việt Nam.

Có thể chọn:

- Sandbox payment gateway.
- VNPay sandbox.
- MoMo test environment.
- Một payment provider có webhook sandbox.

Tôi đề xuất ban đầu tạo interface:

```
public interface IPaymentProvider
{
    Task<PaymentResult> CreatePaymentAsync(
        PaymentRequest request,
        CancellationToken cancellationToken);

    Task<PaymentResult> GetPaymentStatusAsync(
        string transactionId,
        CancellationToken cancellationToken);
}
```

Sau đó:

```
IPaymentProvider
       │
       ├── MockPaymentProvider
       │
       └── RealPaymentProvider
```

Điều này cho phép bạn phát triển payment workflow ngay cả khi chưa tích hợp gateway thật.

### **Payment flow**

Payment GatewayPostgreSQLBackendUserPayment GatewayPostgreSQLBackendUser#chatgpt-mermaid-\_r_52u\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(237, 237, 237);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_52u\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_52u\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_52u\_ .error-icon{fill:rgb(48, 48, 48);}#chatgpt-mermaid-\_r_52u\_ .error-text{fill:rgb(237, 237, 237);stroke:rgb(237, 237, 237);}#chatgpt-mermaid-\_r_52u\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_52u\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_52u\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_52u\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_52u\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_52u\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_52u\_ .marker{fill:rgb(175, 175, 175);stroke:rgb(175, 175, 175);}#chatgpt-mermaid-\_r_52u\_ .marker.cross{stroke:rgb(175, 175, 175);}#chatgpt-mermaid-\_r_52u\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_52u\_ p{margin:0;}#chatgpt-mermaid-\_r_52u\_ .actor{stroke:rgb(31, 78, 148);fill:rgb(9, 23, 44);stroke-width:1;}#chatgpt-mermaid-\_r_52u\_ rect.actor.outer-path[data-look="neo"]{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_52u\_ rect.note[data-look="neo"]{stroke:rgb(58, 132, 63);fill:rgb(48, 48, 48);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_52u\_ text.actor>tspan{fill:rgb(237, 237, 237);stroke:none;}#chatgpt-mermaid-\_r_52u\_ .actor-line{stroke:rgb(175, 175, 175);}#chatgpt-mermaid-\_r_52u\_ .innerArc{stroke-width:1.5;stroke-dasharray:none;}#chatgpt-mermaid-\_r_52u\_ .messageLine0{stroke-width:1.5;stroke-dasharray:none;stroke:rgb(154, 154, 154);}#chatgpt-mermaid-\_r_52u\_ .messageLine1{stroke-width:1.5;stroke-dasharray:2,2;stroke:rgb(154, 154, 154);}#chatgpt-mermaid-\_r_52u\_ [id$="-arrowhead"] path{fill:rgb(154, 154, 154);stroke:rgb(154, 154, 154);}#chatgpt-mermaid-\_r_52u\_ .sequenceNumber{fill:rgb(0, 0, 0);}#chatgpt-mermaid-\_r_52u\_ [id$="-sequencenumber"]{fill:rgb(154, 154, 154);}#chatgpt-mermaid-\_r_52u\_ [id$="-crosshead"] path{fill:rgb(154, 154, 154);stroke:rgb(154, 154, 154);}#chatgpt-mermaid-\_r_52u\_ .messageText{fill:rgb(237, 237, 237);stroke:none;}#chatgpt-mermaid-\_r_52u\_ .labelBox{stroke:rgba(255, 255, 255, 0.15);fill:rgb(0, 0, 0);filter:none;}#chatgpt-mermaid-\_r_52u\_ .labelText,#chatgpt-mermaid-\_r_52u\_ .labelText>tspan{fill:rgb(237, 237, 237);stroke:none;}#chatgpt-mermaid-\_r_52u\_ .loopText,#chatgpt-mermaid-\_r_52u\_ .loopText>tspan{fill:rgb(237, 237, 237);stroke:none;}#chatgpt-mermaid-\_r_52u\_ .sectionTitle,#chatgpt-mermaid-\_r_52u\_ .sectionTitle>tspan{fill:rgb(237, 237, 237);stroke:none;}#chatgpt-mermaid-\_r_52u\_ .loopLine{stroke-width:2px;stroke-dasharray:2,2;stroke:rgba(255, 255, 255, 0.15);fill:rgba(255, 255, 255, 0.15);}#chatgpt-mermaid-\_r_52u\_ .note{stroke:rgb(58, 132, 63);fill:rgb(48, 48, 48);}#chatgpt-mermaid-\_r_52u\_ .noteText,#chatgpt-mermaid-\_r_52u\_ .noteText>tspan{fill:rgb(237, 237, 237);stroke:none;font-weight:normal;}#chatgpt-mermaid-\_r_52u\_ .activation0{fill:rgb(48, 48, 48);stroke:hsl(0, 0%, 8.8235294118%);}#chatgpt-mermaid-\_r_52u\_ .activation1{fill:rgb(48, 48, 48);stroke:hsl(0, 0%, 8.8235294118%);}#chatgpt-mermaid-\_r_52u\_ .activation2{fill:rgb(48, 48, 48);stroke:hsl(0, 0%, 8.8235294118%);}#chatgpt-mermaid-\_r_52u\_ .actorPopupMenu{position:absolute;}#chatgpt-mermaid-\_r_52u\_ .actorPopupMenuPanel{position:absolute;fill:rgb(9, 23, 44);box-shadow:0px 8px 16px 0px rgba(0,0,0,0.2);filter:drop-shadow(3px 5px 2px rgb(0 0 0 / 0.4));}#chatgpt-mermaid-\_r_52u\_ .actor-man circle,#chatgpt-mermaid-\_r_52u\_ line{fill:rgb(9, 23, 44);stroke-width:2px;}#chatgpt-mermaid-\_r_52u\_ g rect.rect{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));stroke:rgb(31, 78, 148);}#chatgpt-mermaid-\_r_52u\_ .node .neo-node{stroke:rgb(31, 78, 148);}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_52u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_52u\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].node .neo-line path{stroke:rgb(31, 78, 148);filter:none;}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_52u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_52u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_52u\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_52u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_52u\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}Confirm checkoutCreate Order + Payment PendingCreate paymentPayment URLPayment URLPayWebhookVerify & update paymentConfirm bookingAcknowledge

### **Các failure cases cần thực hành**

1. User thanh toán thành công nhưng browser đóng.
2. Webhook đến trước khi frontend nhận response.
3. Webhook đến hai lần.
4. Payment gateway timeout.
5. API timeout nhưng gateway vẫn đã nhận request.
6. User refresh checkout.
7. Payment thành công nhưng database update thất bại.
8. Reservation expired trong lúc thanh toán.
9. Gateway gửi callback sai thứ tự.
10. Payment callback có invalid signature.

Bạn không thể đơn giản:

```
if (payment.Success)
{
    reservation.Status = Completed;
}
```

Bạn cần xác định trạng thái giao dịch và xây dựng cơ chế idempotency, consistency và recovery.

## **V4.2 — Idempotency & Payment Failure Recovery**

Đây là một version rất đáng dành riêng.

### **Idempotency**

Client gửi:

```
POST /api/orders
Idempotency-Key: 8f4c...
```

Nếu request bị timeout và client retry cùng key, hệ thống phải trả về kết quả của operation cũ thay vì tạo order thứ hai.

Bạn sẽ nghiên cứu:

- Idempotency key.
- Unique constraints.
- Request fingerprint.
- Retry.
- Duplicate webhook.
- Exactly-once processing vs at-least-once delivery.

Tôi muốn bạn thực hành tư duy rằng:

> Trong hệ thống distributed, "exactly once" thường không phải thứ bạn có thể giả định. Hãy thiết kế xử lý duplicate một cách an toàn.

### **Payment và expiration policy**

Cần chọn một policy rõ ràng.

Ví dụ:

- Reservation giữ ghế 10 phút.
- User bắt đầu thanh toán ở phút thứ 8.
- Payment gateway trả kết quả sau phút thứ 12.

Có thể chọn:

**Policy A:** Reservation vẫn hết hạn đúng deadline, payment thành công muộn sẽ đi vào reconciliation/refund workflow.

**Policy B:** Khi bắt đầu payment, gia hạn reservation một lần theo payment window giới hạn.

Tôi khuyên chọn Policy B cho project demo để UX hợp lý, nhưng vẫn có deadline tuyệt đối, chẳng hạn:

```
Initial hold: 10 minutes
Payment grace: 5 minutes
Maximum hold: 15 minutes
```

Không được gia hạn vô hạn khi user cứ liên tục bắt đầu lại checkout.

# **Phase 5 — Outbox Pattern**

## **V5.0 — Transactional Outbox**

Đây là một trong những phần tôi muốn bạn học kỹ, bởi nó sẽ giúp bạn hiểu reliability trong backend system.

### **Vấn đề**

Backend thực hiện:

```
1. Update reservation
2. Update order
3. Send notification
```

Nếu bước 1 và 2 thành công, bước 3 thất bại:

```
Database: Success
Notification: Failed
```

Nếu bạn retry toàn bộ request có thể gây duplicate hoặc side effect không mong muốn.

### **Outbox Pattern**

Trong cùng transaction:

```
BEGIN

Update Order
Update Payment
Create Ticket
Insert Outbox Message

COMMIT
```

Một background worker xử lý Outbox:

```
Outbox Worker
      ↓
Read pending message
      ↓
Process event
      ↓
Mark processed
```

### **Ví dụ OutboxMessage**

```
Id
EventType
Payload
CreatedAt
ProcessedAt
RetryCount
LastError
```

Event:

```
{
  "eventType": "OrderPaid",
  "orderId": "order-123",
  "userId": "user-456",
  "timestamp": "..."
}
```

### **Những event có thể phát sinh**

```
ReservationCreated
ReservationExpired
OrderCreated
PaymentInitiated
PaymentSucceeded
PaymentFailed
TicketIssued
OrderCancelled
```

Không nhất thiết phải triển khai message broker ngay lập tức. Ở phiên bản đầu, database Outbox + Worker đã đủ để bạn hiểu pattern.

Sau này nếu muốn nâng cấp:

```
Outbox
   ↓
Message Broker
   ↓
Consumers
```

Có thể thêm RabbitMQ hoặc Kafka ở một version advanced.

### **Acceptance Criteria**

- Outbox record được ghi cùng transaction với business operation.
- Worker đọc và xử lý pending messages.
- Worker crash không mất event.
- Retry được event thất bại.
- Duplicate event không tạo duplicate ticket.
- Có thể simulate lỗi giữa các bước xử lý.
- Có log và status cho Outbox.

# **Phase 6 — Blazor WebAssembly + MVVM**

Tôi đề xuất bắt đầu frontend ngay từ V1 để bạn không phải làm một frontend khổng lồ ở cuối project. Nhưng ở Phase 6, bạn sẽ refactor và hoàn thiện kiến trúc MVVM rõ ràng.

## **V6.0 — Blazor Fundamentals**

Học:

- Component.
- Parameters.
- Event callbacks.
- Binding.
- Lifecycle.
- Routing.
- Dependency injection.
- HTTP client.
- Async UI.
- Loading state.
- Error state.

Pages:

```
/
├── Event Catalog
├── Event Detail
├── Seat Selection
├── Checkout
├── Payment Result
└── My Tickets
```

## **V6.1 — MVVM Architecture**

Ví dụ:

```
SeatSelectionPage.razor
         ↓
SeatSelectionViewModel
         ↓
ISeatBookingService
         ↓
HttpClient
         ↓
ASP.NET Core API
```

### **View**

```
@page "/events/{EventId}/seats"
@inject SeatSelectionViewModel ViewModel

@if (ViewModel.IsLoading)
{
    <LoadingSpinner />
}
else
{
    <SeatMap Seats="ViewModel.Seats"
             SelectedSeats="ViewModel.SelectedSeats"
             OnSeatSelected="ViewModel.SelectSeat" />
}
```

### **ViewModel**

```
public class SeatSelectionViewModel
{
    public bool IsLoading { get; private set; }

    public List<SeatViewModel> Seats { get; private set; } = [];

    public List<SeatViewModel> SelectedSeats { get; private set; } = [];

    public async Task LoadAsync(Guid eventId)
    {
        IsLoading = true;

        try
        {
            // Load seat map
        }
        finally
        {
            IsLoading = false;
        }
    }

    public void SelectSeat(SeatViewModel seat)
    {
        // Selection logic
    }
}
```

Đây mới là phần cơ bản. Khi UI thay đổi do ViewModel cập nhật, bạn sẽ cần cơ chế notification phù hợp để View render lại.

### **MVVM implementation choices**

Bạn có thể triển khai MVVM thuần bằng:

- `INotifyPropertyChanged`
- `ObservableObject`
- `CommunityToolkit.Mvvm`

Tôi đề xuất ban đầu tự implement một ViewModel nhỏ với `INotifyPropertyChanged`, để hiểu cơ chế. Sau đó thử CommunityToolkit.Mvvm và so sánh.

## **V6.2 — Interactive Seat Map**

Đây là feature quan trọng nhất của frontend.

[Seats booking interactions by Den Klenkov on Dribbble](https://images.openai.com/static-rsc-4/WT6GFZ_4UMdlXdsiSOWIVJrwJ8xnIH1Y1iS9zQOub8hpT_iFIdqlmDy4GRclQ12OAIcYy6Bj77a_V1pVLCssWPRqmJ7jqpdhrQErmb2ynJdH7Uaqol-ZhDJwShynQ4Fr2MnzUtvblGFd5juMNy6vvcYXG5lctyf3Faj1XovgBOc?purpose=inline)

[Management Software for Auditorium | FanGo](https://images.openai.com/static-rsc-4/S03xUq2Rlc8M_OYPfjI3u53q8vzWzAKMML8cmEiV-fVC-jwAvEI4gik_Z_23J7mcwxFQfxMDzlE5qFfQOthN65T4_wKXP6YCs57vrNQFO_8pAO7QNeab2B_rBQ1lxw2tO3NL5vYM5T_9wlFCJzeBc-f4XfSLKiZJh3uYzT90iPk?purpose=inline)

[Cara Memesan Tiket Bioskop Melalui M-Tix – cK stuff](https://images.openai.com/static-rsc-4/Oqfjty89M-wqJfs5aOw_FPennl4mcOyqyFdlofslQVXZWBCDKWLNJh5W6CRUZ94FOGLUIwfMdbqHYejyJumJFI5Drm9ovD2kQPzWllgi_I9EnsbNAOxpTUx-LlUh_Wox8FIAGKQwggIQ6pZm1-urYPlimhNpoj8JIjX1IyfBGGc?purpose=inline)

12

Bạn cần xây dựng:

- Seat rows.
- Sections.
- VIP seats.
- Regular seats.
- Disabled seats.
- Sold seats.
- Temporarily reserved seats.
- Selected seats.
- Zoom/pan.
- Tooltip.
- Seat price.
- Selected seat summary.
- Total amount.

### **State mapping**

| Backend state            | UI              |
| ------------------------ | --------------- |
| Available                | Có thể chọn     |
| Reserved by current user | Selected / Held |
| Reserved by other user   | Disabled        |
| Sold                     | Disabled        |
| Disabled                 | Disabled        |

Lưu ý UI không được coi trạng thái hiện tại là sự thật tuyệt đối. Khi người dùng bấm chọn ghế, server vẫn phải kiểm tra lại.

## **V6.3 — Real-time Seat Availability**

Sau khi seat map cơ bản hoạt động, thêm real-time.

Tôi đề xuất **SignalR**.

Ví dụ:

```
User A reserves A12
        ↓
Backend commits database
        ↓
SignalR event
        ↓
User B receives update
        ↓
A12 becomes unavailable
```

Events:

```
SeatReserved
SeatReleased
SeatSold
```

SignalR chỉ thông báo state change. Nó không thay thế PostgreSQL transaction.

Nếu client disconnect:

```
Reconnect
    ↓
Fetch latest seat map
    ↓
Reconcile UI state
```

Không nên tin rằng client đã nhận đủ event trong lúc mất kết nối.

# **Phase 7 — Testing & Reliability**

Tôi muốn project này có test nhiều hơn các project trước, vì transactional project không thể chứng minh tính đúng đắn chỉ bằng việc thử click trên UI.

## **V7.0 — Unit Tests**

Test Domain rules:

```
Cannot reserve sold seat
Cannot reserve disabled seat
Reservation expires correctly
Cannot cancel completed reservation
Cannot complete expired reservation
Cannot create negative payment amount
```

Sử dụng xUnit và một mocking library nếu cần, nhưng không mock mọi thứ một cách máy móc.

## **V7.1 — Integration Tests**

Kiểm tra:

- PostgreSQL.
- EF Core.
- Transactions.
- Redis.
- API endpoints.
- Authentication.
- Payment workflows.

Tôi khuyên sử dụng Testcontainers để tạo database và Redis thực trong test environment thay vì dựa hoàn toàn vào mocks.

## **V7.2 — Concurrency Tests**

Đây là test suite riêng của project.

Ví dụ:

```
var tasks = Enumerable.Range(0, 100)
    .Select(_ => bookingService.ReserveAsync(
        userId: Guid.NewGuid(),
        seatId: targetSeatId));

var results = await Task.WhenAll(tasks);
```

Assertions:

```
Successful bookings = 1
Duplicate booking = 0
Seat final state = Reserved or Sold
```

Ngoài ra:

- 100 users × 10 seats.
- Multiple seat requests.
- Random seat selections.
- Concurrent cancellation.
- Concurrent reservation expiration.
- Payment callback concurrent requests.

## **V7.3 — Fault Injection**

Cố tình gây lỗi:

```
Redis unavailable
Database timeout
Payment timeout
Worker crash
Webhook duplicate
Network interruption
```

Mục tiêu:

> Không phải hệ thống không bao giờ lỗi, mà là khi lỗi xảy ra, hệ thống không tạo ra dữ liệu sai và có khả năng recovery.

# **Phase 8 — Security, Observability & Deployment**

## **V8.0 — Security**

Các vấn đề nên xử lý:

- Authentication.
- Authorization.
- Ownership validation.
- Input validation.
- Rate limiting.
- Idempotency.
- Webhook signature validation.
- SQL injection.
- Secret management.
- CORS.
- Logging không chứa thông tin payment nhạy cảm.
- Token storage cho Blazor WASM.
- OWASP basics.

## **V8.1 — Observability**

Thêm:

- Structured logging.
- Correlation ID.
- Request tracing.
- Metrics.
- Health checks.
- Worker monitoring.

Các metric có thể quan sát:

```
reservation.success
reservation.failure
reservation.expired
payment.success
payment.failure
seat.contention
outbox.pending
outbox.failed
worker.processing_time
```

Một dashboard có thể hiển thị:

- Số reservation đang active.
- Reservation expire rate.
- Payment success rate.
- Booking conflict rate.
- Outbox backlog.
- API latency.

Có thể thực hành OpenTelemetry và Prometheus/Grafana ở version này.

## **V8.2 — Deployment**

Đóng gói:

```
Blazor WASM
ASP.NET Core API
Worker
PostgreSQL
Redis
```

Deploy một môi trường demo, có:

- HTTPS.
- Environment configuration.
- Database migration.
- Health endpoint.
- CI/CD.
- README.
- Architecture diagram.
- Demo account.

# **6. Roadmap rút gọn**

### **Event Booking — Project Roadmap**

| **Version** | **Mục tiêu**         | **Trọng tâm**                                        |
| ----------- | -------------------- | ---------------------------------------------------- |
| V0          | Foundation           | Solution, Docker, PostgreSQL, Redis                  |
| V1          | Event Management     | CRUD, EF Core, Auth                                  |
| V2          | Booking Core         | Seat map, reservation, PostgreSQL locks              |
| V3          | Expiration           | Redis TTL, background worker                         |
| V4          | Order & Payment      | Payment lifecycle, idempotency, recovery             |
| V5          | Reliable Events      | Outbox pattern                                       |
| V6          | Frontend             | Blazor WASM, MVVM, interactive seat map              |
| V7          | Testing              | Unit, integration, concurrency, fault injection      |
| V8          | Production readiness | Security, observability, deployment                  |
| V9          | Advanced             | Load test, scaling, broker, architecture experiments |

# **7. Tôi sẽ thay đổi một chút thứ tự học**

Bạn đề xuất:

> ASP.NET Core + PostgreSQL locking + Redis TTL + Outbox + Blazor MVVM.

Tôi đồng ý toàn bộ stack. Nhưng tôi sẽ không học tất cả cùng lúc.

Thứ tự dependency của kiến thức nên là:

1. **Domain & Database** — Entities, relationships, constraints
2. **Transactions** — ACID, rollback, isolation levels
3. **Concurrency** — Race condition, locking, optimistic concurrency
4. **Reservation** — Atomic multi-seat booking
5. **Redis** — TTL, temporary state, cache
6. **Background processing** — Expiration worker, retry
7. **Order & Payment** — State machine, idempotency
8. **Outbox** — Transactional event publishing
9. **Blazor + MVVM** — UI architecture
10. **SignalR** — Live seat availability
11. **Reliability & deployment** — Testing, observability, load testing

Một lý do quan trọng: bạn nên hiểu backend booking và các invariants trước khi UI realtime xuất hiện. Nếu không, UI có thể trông rất hoàn thiện nhưng backend vẫn không đảm bảo được seat ownership.

# **8. Những điểm tôi khuyên bạn không nên làm ngay**

Project này có thể rất dễ scope creep. Tôi muốn giữ nó challenging nhưng vẫn hoàn thành được.

### **Chưa làm microservices**

Không cần tách:

```
Booking Service
Payment Service
Event Service
Ticket Service
Notification Service
```

ngay từ đầu.

Thay vào đó, modular monolith, nhưng boundaries rõ ràng. Khi đã hoàn thành, bạn có thể thử tách Booking hoặc Payment ra service riêng như một bài advanced.

### **Không tích hợp payment thật từ V1**

Bắt đầu với `MockPaymentProvider`, tạo khả năng mô phỏng:

- Success.
- Failure.
- Timeout.
- Delayed callback.
- Duplicate webhook.

Khi workflow hoàn chỉnh mới kết nối sandbox provider.

### **Không xây seat map quá phức tạp ngay**

Ban đầu seat map dạng grid:

```
        STAGE

 A  [1] [2] [3] [4] [5]

 B  [1] [2] [3] [4] [5]

 C  [1] [2] [3] [4] [5]
```

Sau đó nâng cấp thành sections, SVG layout, zoom, pan, VIP zones.

### **Không thêm Kafka chỉ để có Kafka**

Outbox Pattern với PostgreSQL + Worker đã giải quyết một bài toán thực tế và giúp bạn học rất nhiều. Broker là bước nâng cấp, không phải prerequisite.

### **Không để Redis quyết định booking correctness**

Nếu Redis unavailable, hệ thống có thể tạm ngừng một số chức năng hoặc degradation tùy thiết kế, nhưng database vẫn phải bảo đảm invariant.

# **9. Đánh giá project này đối với portfolio của bạn**

Tôi đánh giá project này cao hơn GoldWatch về chiều sâu backend engineering, nếu bạn hoàn thiện được các phần quan trọng.

| Tiêu chí                | GoldWatch | Event Booking |
| ----------------------- | --------- | ------------- |
| CRUD & API              | ★★★★☆     | ★★★★☆         |
| Database design         | ★★★★☆     | ★★★★★         |
| Business logic          | ★★★★☆     | ★★★★★         |
| Concurrency             | ★★☆☆☆     | ★★★★★         |
| Transaction             | ★★★☆☆     | ★★★★★         |
| Distributed systems     | ★★★☆☆     | ★★★★★         |
| Frontend architecture   | ★★★★☆     | ★★★★☆         |
| Testing                 | ★★★☆☆     | ★★★★★         |
| Backend interview value | ★★★★☆     | ★★★★★         |

Nếu thực hiện đến V7–V8, bạn sẽ có nhiều chủ đề để trình bày khi phỏng vấn .NET Backend:

- Why did you choose PostgreSQL row-level locking?
- How do you prevent double booking?
- What happens when two users reserve the same seat?
- How do you handle payment webhook duplication?
- Why do you need the Outbox Pattern?
- What happens when Redis goes down?
- How do you ensure reservation expiration?
- How do you handle deadlocks?
- How do you recover from failed background jobs?
- How do you test concurrency?
- How do you design idempotent APIs?

Đây không chỉ là câu hỏi lý thuyết nữa, bởi bạn sẽ có project và test thực tế để chứng minh.

## **Định hướng cuối cùng**

Tôi sẽ định vị project này như sau:

> **EventBooking — A transactional event ticketing platform built with ASP.NET Core, PostgreSQL, Redis and Blazor WebAssembly, featuring concurrency-safe seat reservations, payment workflows, expiration handling and reliable event processing.**

Và tôi muốn roadmap này đi theo phong cách chúng ta từng làm với SchoolCRUD và GoldWatch: **mỗi version có mục tiêu cụ thể, kiến thức cần nghiên cứu, task, cấu trúc code, acceptance criteria và các bài kiểm tra để xác minh bạn thực sự hiểu phần vừa làm.**

Bước tiếp theo hợp lý nhất là bắt đầu **V0.1 — Environment & Solution Foundation**, thiết lập solution, Docker PostgreSQL/Redis, Web API và Blazor WASM, sau đó xây dựng project từng version thay vì code toàn bộ kiến trúc một lần.

Guide EventBooking V0.1