# 🎯 QuizGame – Kvíz játék

**MFW beadandó** · Kiss István (T73N0W) · 2025/26 tavaszi félév

REST architektúrájú webalkalmazás online kvízek létrehozására, kezelésére és valós idejű lebonyolítására.

---

## 📑 Tartalomjegyzék

1. [Feladatleírás](#1-feladatleírás)
2. [Funkcionális elemzés](#2-funkcionális-elemzés)
3. [Rendszerarchitektúra](#3-rendszerarchitektúra)
4. [Statikus terv](#4-statikus-terv)
5. [Adatbázis terv](#5-adatbázis-terv)
6. [Szolgáltatás réteg](#6-szolgáltatás-réteg)
7. [Webszolgáltatás réteg](#7-webszolgáltatás-réteg)
8. [SignalR kommunikáció](#8-signalr-kommunikáció)
9. [Frontend felépítése](#9-frontend-felépítése)
10. [Tesztelés](#10-tesztelés)
11. [Képernyőképek és demó](#11-képernyőképek-és-demó)

---

## 1. Feladatleírás

A projekt célja egy REST architektúrájú webalkalmazás elkészítése volt, amely lehetőséget biztosít online kvízek létrehozására, kezelésére és valós idejű lebonyolítására.

Az alkalmazás két fő felületből áll:

- **Adminisztrációs felület:** a bejelentkezett felhasználók saját kvízeket hozhatnak létre, módosíthatnak, valamint vezérelhetik azok lebonyolítását.
- **Résztvevői felület:** a felhasználók PIN kód segítségével csatlakozhatnak aktív kvízekhez, és valós időben válaszolhatnak a kérdésekre.

---

## 2. Funkcionális elemzés

### 2.1 Felhasználói szerepkörök

| Szerepkör | Lehetőségek |
|---|---|
| **Adminisztrátor** (kvíz tulajdonos) | regisztráció, bejelentkezés, saját kvízek létrehozása / megtekintése / szerkesztése / törlése, kvíz megnyitása és indítása, léptetés a kérdések között, aktuális kérdés lezárása, aktív kvíz befejezése |
| **Résztvevő** | csatlakozás PIN kóddal, aktuális kérdés megtekintése, válasz beküldése, eredmény megtekintése |

### 2.2 Fő funkcionalitások

#### Felhasználókezelés

- regisztráció
- bejelentkezés
- JWT alapú autentikáció
- refresh token alapú token megújítás
- kijelentkezés

#### Kvízkezelés

A bejelentkezett felhasználók:

- új kvízt hozhatnak létre,
- megtekinthetik saját kvízeiket,
- törölhetik vagy módosíthatják a még el nem indított kvízeket.

Egy kvíz:

- legalább egy kérdést tartalmaz,
- minden kérdés legalább két válaszlehetőséget tartalmaz,
- minden kérdéshez pontosan egy helyes válasz tartozik.

#### Kvíz lebonyolítás (session kezelés)

A rendszer a kvízek futtatását külön sessionökkel kezeli. A session állapotai: `Waiting`, `InProgress`, `Finished`.

```mermaid
stateDiagram-v2
    [*] --> Waiting: Session megnyitása
    Waiting --> InProgress: Indítás
    InProgress --> InProgress: Kérdés lezárása / következő kérdés
    InProgress --> Finished: Befejezés
    Finished --> [*]
```

Az adminisztrátor:

- elindíthatja a sessiont,
- lezárhatja az aktuális kérdést,
- következő kérdésre léphet,
- befejezheti a sessiont.

#### Valós idejű kommunikáció

A SignalR segítségével:

- a résztvevők automatikusan megkapják az aktuális kérdést,
- az állapotváltozások azonnal megjelennek,
- nincs szükség az oldal újratöltésére.

---

## 3. Rendszerarchitektúra

### 3.1 Technológiák

| Terület | Technológiák |
|---|---|
| **Backend** | ASP.NET Core Web API, Entity Framework Core, ASP.NET Identity, JWT Authentication, SignalR, AutoMapper |
| **Frontend** | React, React Router, Bootstrap |
| **Adatbázis** | Microsoft SQL Server |
| **Tesztelés** | xUnit, ASP.NET Core Integration Testing |

---

## 4. Statikus terv

### 4.1 Réteges architektúra

A rendszer többrétegű architektúrát használ.

**Fő komponensek:**

- Frontend kliens alkalmazás
- WebAPI réteg
- Szolgáltatás réteg
- Adatkezelési réteg
- SQL adatbázis
- SignalR kommunikációs réteg

**A projekt fizikai felépítése:**

```text
QuizGame
├── QuizGame.DataAccess
├── QuizGame.Shared
├── QuizGame.SignalR
├── QuizGame.WebAPI
└── QuizGame.React
```

### 4.2 UML komponens diagram

```mermaid
flowchart TB
    DB[("mssql server db")]
    DA["EF Core / Services<br/>(QuizGame.DataAccess)"]
    Shared["Shared Dtos<br/>(QuizGame.Shared)"]
    API["ASP.NET Core REST API<br/>(QuizGame.WebAPI)"]
    React["React frontend<br/>(QuizGame.React)"]
    SignalR["Real time messaging<br/>(QuizGame.SignalR)"]
    Tests["Integration tests<br/>(QuizGame.Tests)"]

    DA -- "SQL queries" --> DB
    DA --> Shared
    API -- "Uses Dtos" --> Shared
    React -- "HTTP/JSON" --> API
    API --> SignalR
    Tests --> Shared
    Tests --> API
```

---

## 5. Adatbázis terv

### 5.1 Adatbázis leírás

Az alkalmazás relációs adatbázist használ Entity Framework Core ORM segítségével. Az alábbi ábra az adatbázis sémáját mutatja.

```mermaid
erDiagram
    Quizzes {
        int Id PK
        string Title
        string UserId
        datetime CreatedAt
        datetime DeletedAt
        bool IsPublished
        string PinCode
        string Status
    }
    Questions {
        int Id PK
        int QuizId FK
        string Text
        int Order
        int TimeLimitSeconds
    }
    AnswersOptions {
        int Id PK
        int QuestionId FK
        string Text
        bool IsCorrect
    }
    QuizSessions {
        int Id PK
        int QuizId FK
        datetime StartedAt
        datetime EndedAt
        int CurrentQuestionId FK
        bool IsQuestionLocked
        string State
    }
    Participants {
        int Id PK
        int QuizSessionId FK
        string UserId FK
        string Nickname
        datetime JoinedAt
    }
    Submissions {
        int Id PK
        int ParticipantId FK
        int QuestionId FK
        int AnswerOptionId FK
        datetime SubmittedAt
        bool IsCorrect
    }
    AspNetUsers {
        string Id PK
        string Name
        string RefreshToken
        string UserName
        string NormalizedUserName
        string Email
        string NormalizedEmail
        bool EmailConfirmed
        string PasswordHash
        string SecurityStamp
        string ConcurrencyStamp
        string PhoneNumber
        bool PhoneNumberConfirmed
        bool TwoFactorEnabled
        datetime LockoutEnd
        bool LockoutEnabled
        int AccessFailedCount
    }
    AspNetUserRoles {
        string UserId PK
        string RoleId PK
    }

    Quizzes ||--o{ Questions : "tartalmaz"
    Questions ||--o{ AnswersOptions : "tartalmaz"
    Quizzes ||--o{ QuizSessions : "sessionjei"
    QuizSessions }o--o| Questions : "aktuális kérdés"
    QuizSessions ||--o{ Participants : "résztvevői"
    AspNetUsers ||--o{ Participants : "csatlakozik"
    Participants ||--o{ Submissions : "leadja"
    Questions ||--o{ Submissions : "erre a kérdésre"
    AnswersOptions ||--o{ Submissions : "ezt választották"
    AspNetUsers ||--o{ AspNetUserRoles : "szerepkörei"
```

### 5.2 Kapcsolatok

| Kapcsolat | Leírás |
|---|---|
| Quiz – Question | Egy kvíz több kérdést tartalmaz. Egy kérdés pontosan egy kvízhez tartozik. |
| Question – AnswerOption | Egy kérdés több válaszlehetőséget tartalmaz. |
| Quiz – QuizSession | Egy kvízhez több session tartozhat. |
| QuizSession – Participant | Egy sessionhöz több résztvevő tartozhat. |
| Participant – Submission | Egy résztvevő több választ adhat le. |
| AnswerOption – Submission | Egy válaszlehetőséghez több leadott beküldés is tartozhat. |

---

## 6. Szolgáltatás réteg

A szolgáltatás rétegben találhatók a közvetlen adatbázis-műveletek. A réteg a `QuizGame.DataAccess` projektben valósult meg.

### Szolgáltatások osztálydiagramokban

#### Kvíz szolgáltatás

```mermaid
classDiagram
    class IQuizService {
        <<interface>>
    }
    class QuizService {
        -QuizGameDbContext _context
        +QuizService(QuizGameDbContext context)
        +GetAllAsync() Task~List~Quiz~~
        +GetByIdAsync(int id) Task~Quiz?~
        +CreateAsync(Quiz quiz) Task~Quiz~
        +UpdateAsync(Quiz quiz) Task
        +DeleteAsync(int id) Task
    }
    IQuizService <|-- QuizService
```

#### Kérdés szolgáltatás

```mermaid
classDiagram
    class IQuestionService {
        <<interface>>
    }
    class QuestionService {
        -QuizGameDbContext _context
        +QuestionService(QuizGameDbContext context)
        +GetByQuizIdAsync(int quizId) Question[]
        +GetByIdAsync(int id) Question
        +CreateAsync(Question question) Question
        +UpdateAsync(Question question)
        +DeleteAsync(int id)
    }
    IQuestionService <|-- QuestionService
```

#### Kvíz session szolgáltatás

```mermaid
classDiagram
    class IQuizSessionService {
        <<interface>>
    }
    class QuizSessionService {
        -QuizGameDbContext _context
        +QuizSessionService(QuizGameDbContext context)
        +GetByIdAsync(int id) QuizSession
        +GetByQuizIdAsync(int quizId) QuizSession[]
        +LockQuestionAsync(int sessionId)
        +NextQuestionAsync(int sessionId)
        +CreateAsync(int quizId) QuizSession
        +StartSessionAsync(int sessionId)
        +SetCurrentQuestionAsync(int sessionId, int questionId)
        +CloseSessionAsync(int sessionId)
    }
    IQuizSessionService <|-- QuizSessionService
```

#### Felhasználó szolgáltatás

```mermaid
classDiagram
    class IUsersService {
        <<interface>>
    }
    class UsersService {
        -JwtSettings _jwtSettings
        -IHttpContextAccessor _httpContextAccessor
        -UserManager _userManager
        -SignInManager _signInManager
        +UsersService(IOptions jwtSettings, IHttpContextAccessor httpContextAccessor, UserManager userManager, SignInManager signInManager)
        +AddUserAsync(User user, string password, Role role)
        +LoginAsync(string email, string password) AuthResult
        +RedeemRefreshTokenAsync(string refreshToken) AuthResult
        +LogoutAsync()
        +GetCurrentUserAsync() User
        +GetCurrentUserId() string
        +GetUserByIdAsync(string id) User
        +GetCurrentUserRoles() Role[]
        +IsCurrentUserAdmin() bool
        -GenerateJwtTokenAsync(User user) string
    }
    IUsersService <|-- UsersService
```

#### Résztvevő szolgáltatás

```mermaid
classDiagram
    class IParticipantService {
        <<interface>>
    }
    class ParticipantService {
        -QuizGameDbContext _context
        -UserManager _userManager
        +ParticipantService(QuizGameDbContext context)
        +JoinAsync(int sessionId, string nickname, string userId)
        +GetBySessionAsync(int sessionId) Participant[]
        +GetByIdAsync(int id) Participant
    }
    IParticipantService <|-- ParticipantService
```

#### Leadás szolgáltatás

```mermaid
classDiagram
    class ISubmissionService {
        <<interface>>
    }
    class SubmissionService {
        -QuizGameDbContext _context
        +SubmissionService(QuizGameDbContext context)
        +SubmitAnswerAsync(int participantId, int questionId, int answerOptionId) Task~Submission~
        +GetByQuestionAsync(int questionId) Task~List~Submission~~
        +GetByParticipantAsync(int participantId) Task~List~Submission~~
        +GetByParticipantAndQuestionAsync(int participantId, int questionId) Task~Submission?~
    }
    ISubmissionService <|-- SubmissionService
```

---

## 7. Webszolgáltatás réteg

### 7.1 QuizController

```mermaid
classDiagram
    class ControllerBase
    class QuizController {
        -IQuizService _quizService
        -IQuizSessionService _sessionService
        -IMapper _mapper
        -UserManager~User~ _userManager
        +QuizController(IQuizService quizService, IQuizSessionService sessionService, IMapper mapper, UserManager~User~ userManager)
        +GetAll() Task~ActionResult~QuizzesResponseDto~~
        +GetMyQuizzes() Task~ActionResult~List~QuizListItemDto~~~
        +GetById(int id) Task~ActionResult~QuizDetailsDto~~
        +Create(CreateQuizDto model) Task~ActionResult~
        +Update(int id, UpdateQuizDto model) Task~IActionResult~
        +Delete(int id) Task~IActionResult~
    }
    ControllerBase <|-- QuizController
```

**Feladata:** kvízek kezelése, lekérdezése, létrehozása, módosítása, törlése.

| Metódus | Végpont | Leírás |
|---|---|---|
| GET | `/api/quiz` | Összes kvíz lekérdezése |
| GET | `/api/quiz/{id}` | Kvíz lekérdezése azonosító alapján |
| POST | `/api/quiz` | Kvíz létrehozása |
| PUT | `/api/quiz/{id}` | Kvíz módosítása |
| DELETE | `/api/quiz/{id}` | Kvíz törlése |

### 7.2 UsersController

```mermaid
classDiagram
    class ControllerBase
    class UsersController {
        -IUsersService _usersService
        -IMapper _mapper
        +UsersController(IMapper mapper, IUsersService usersService)
        +CreateUser(UserRequestDto userRequestDto) Task~IActionResult~
        +GetUser(string id) Task~IActionResult~
        +Login(LoginRequestDto loginRequestDto) Task~IActionResult~
        +Logout() Task~IActionResult~
        +RedeemRefreshToken(string refreshToken) Task~IActionResult~
    }
    ControllerBase <|-- UsersController
```

**Feladata:** felhasználókezelés, autentikáció, token kezelés.

| Metódus | Végpont | Leírás |
|---|---|---|
| POST | `/api/user/register` | Regisztráció |
| POST | `/api/user/login` | Bejelentkezés |
| POST | `/api/user/logout` | Kijelentkezés |
| POST | `/api/user/refresh` | Token megújítása |
| GET | `/api/user/{id}` | Felhasználó lekérdezése |

### 7.3 SessionController

```mermaid
classDiagram
    class ControllerBase
    class SessionController {
        -IQuizSessionService _sessionService
        -IQuizService _quizService
        -ISubmissionService _submissionService
        -UserManager~User~ _userManager
        -IMapper _mapper
        -IHubContext~QuizHub~ _hub
        +SessionController(IQuizSessionService sessionService, IQuizService quizService, ISubmissionService submissionService, UserManager~User~ userManager, IMapper mapper, IHubContext~QuizHub~ hub)
        +Open(int quizId) Task~ActionResult~int~~
        +Get(int sessionId, int? participantId) Task~ActionResult~QuizSessionDto~~
        +Start(int sessionId) Task~IActionResult~
        +Lock(int sessionId) Task~IActionResult~
        +Next(int sessionId) Task~IActionResult~
        +End(int sessionId) Task~IActionResult~
        -BroadcastSessionUpdate(int sessionId) Task
    }
    ControllerBase <|-- SessionController
```

**Feladata:** sessionök kezelése, a kvíz lebonyolításának vezérlése.

| Metódus | Végpont | Leírás |
|---|---|---|
| POST | `/api/session/open/{quizId}` | Session megnyitása |
| POST | `/api/session/{sessionId}/start` | Session indítása |
| POST | `/api/session/{sessionId}/lock` | Aktuális kérdés lezárása |
| POST | `/api/session/{sessionId}/next` | Következő kérdés |
| POST | `/api/session/{sessionId}/end` | Session befejezése |
| GET | `/api/session/{sessionId}` | Session állapotának lekérdezése |

### 7.4 PlayerController

```mermaid
classDiagram
    class ControllerBase
    class PlayerController {
        -IQuizService _quizService
        -IQuizSessionService _sessionService
        -IParticipantService _participantService
        -ISubmissionService _submissionService
        -IHubContext~QuizHub~ _hub
        +PlayerController(IQuizService quizService, IQuizSessionService sessionService, IParticipantService participantService, ISubmissionService submissionService, IHubContext~QuizHub~ hub)
        +Join(JoinQuizRequestDto model) Task~ActionResult~JoinQuizResponseDto~~
        +Submit(SubmitAnswerDto model) Task~IActionResult~
        -BroadcastSessionUpdate(int sessionId) Task
    }
    ControllerBase <|-- PlayerController
```

**Feladata:** résztvevők csatlakozása, válaszok beküldése.

| Metódus | Végpont | Leírás |
|---|---|---|
| POST | `/api/player/join` | Csatlakozás PIN kóddal |
| POST | `/api/player/submit` | Válasz beküldése |

---

## 8. SignalR kommunikáció

### 8.1 QuizHub

A rendszer SignalR Hub segítségével biztosítja a valós idejű kommunikációt. A kliensek session csoportokhoz csatlakoznak.

Az alábbi állapotváltozások automatikusan továbbításra kerülnek minden résztvevő számára:

- kérdés aktiválása
- kérdés lezárása
- következő kérdés
- session befejezése

```mermaid
sequenceDiagram
    participant A as Admin
    participant C as SessionController
    participant H as QuizHub
    participant P as Résztvevők

    P->>H: Csatlakozás a session csoporthoz
    A->>C: POST /api/session/{id}/start
    C->>H: BroadcastSessionUpdate
    H-->>P: Aktuális kérdés
    P->>C: POST /api/player/submit
    A->>C: POST /api/session/{id}/lock
    C->>H: BroadcastSessionUpdate
    H-->>P: Kérdés lezárva
    A->>C: POST /api/session/{id}/next
    C->>H: BroadcastSessionUpdate
    H-->>P: Következő kérdés
    A->>C: POST /api/session/{id}/end
    C->>H: BroadcastSessionUpdate
    H-->>P: Session befejezve
```

---

## 9. Frontend felépítése

### 9.1 Adminisztrációs felület

Főbb oldalak:

- `Login`
- `Register`
- `MyQuizzes`
- `QuizDetails`
- `CreateQuiz`
- `EditQuiz`
- `Session` oldal

### 9.2 Résztvevői felület

Főbb funkciók:

- csatlakozás PIN kóddal
- aktuális kérdés megjelenítése
- válaszadás
- eredmények megtekintése

---

## 10. Tesztelés

### 10.1 Integrációs tesztek

A webszolgáltatás funkcionalitásának ellenőrzése integrációs tesztekkel történt (xUnit + ASP.NET Core Integration Testing).

### 10.2 Tesztelt komponensek

#### QuizController tesztek

```mermaid
classDiagram
    class IClassFixture
    class QuizControllerTests {
        -TestWebApplicationFactory _factory
        -HttpClient _client
        +QuizControllerTests(TestWebApplicationFactory factory)
        +GetAll_ReturnsOk()
        +GetById_ReturnsNotFound_ForInvalidId()
        +GetById_ReturnsQuiz_WhenQuizExists()
        +Create_ReturnsUnauthorized_WithoutToken()
        +Delete_ReturnsUnauthorized_WithoutToken()
        +Create_ReturnsCreated_WithValidToken()
        +Delete_ReturnsNoContent_WithValidToken()
        -RegisterAndLogin() string
    }
    IClassFixture <|-- QuizControllerTests
```

Tesztelt esetek:

- összes kvíz lekérdezése
- kvíz lekérdezése azonosító alapján
- nem létező kvíz kezelése
- kvíz létrehozása
- kvíz létrehozása autorizáció nélkül
- jogosultság ellenőrzése
- kvíz törlése

#### UsersController tesztek

```mermaid
classDiagram
    class IClassFixture
    class UsersControllerTests {
        -HttpClient _client
        +UsersControllerTests(TestWebApplicationFactory factory)
        +Register_ReturnsCreated_WhenValid()
        +Register_ReturnsConflict_WhenEmailAlreadyExists()
        +Login_ReturnsOk_WhenCredentialsValid()
        +Login_ReturnsForbidden_WhenPasswordInvalid()
        +GetUser_ReturnsUnauthorized_WithoutToken()
        +GetUser_ReturnsOk_WithValidToken()
    }
    IClassFixture <|-- UsersControllerTests
```

Tesztelt esetek:

- regisztráció
- bejelentkezés
- hibás bejelentkezés
- felhasználó lekérdezése
- jogosultság kezelés

#### SessionController tesztek

```mermaid
classDiagram
    class IClassFixture
    class SessionControllerTests {
        -TestWebApplicationFactory _factory
        -HttpClient _client
        +SessionControllerTests(TestWebApplicationFactory factory)
        -LoginAndGetToken() string
        +Get_ReturnsNotFound_ForInvalidSession()
        +Get_ReturnsSession_WhenSessionExists()
        +Open_ReturnsUnauthorized_WithoutToken()
        +Open_ReturnsOk_WithValidToken()
        +Start_ReturnsNoContent_WhenSessionExists()
        +Lock_ReturnsNoContent_WhenSessionExists()
        +End_ReturnsNoContent_WhenSessionExists()
    }
    IClassFixture <|-- SessionControllerTests
```

Tesztelt esetek:

- session megnyitása
- session megnyitása nem létező session esetén
- session indítása autentikáció nélkül
- session indítása
- kérdés lezárása
- session befejezése
- session állapot lekérdezése

#### PlayerController tesztek

```mermaid
classDiagram
    class IClassFixture
    class PlayerControllerTests {
        -TestWebApplicationFactory _factory
        -HttpClient _client
        +PlayerControllerTests(TestWebApplicationFactory factory)
        +Join_ReturnsOk_WhenValid()
        +Join_ReturnsBadRequest_WhenPinInvalid()
        +Join_ReturnsBadRequest_WhenQuizAlreadyStarted()
        +Submit_ReturnsNoContent_WhenValid()
        +Submit_ReturnsBadRequest_WhenParticipantInvalid()
    }
    IClassFixture <|-- PlayerControllerTests
```

Tesztelt esetek:

- csatlakozás PIN kóddal
- hibás PIN kezelés
- csatlakozás elkezdett sessionhöz
- válasz beküldése
- helytelen résztvevőként válaszolás

---

## 11. Képernyőképek és demó

### 📋 TODO
