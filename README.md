# RaceDay-API
ASP.NET Core (.NET 10) Web API for managing RACEDAY users, events, categories, enrolments and results.
Entity Framework Core + SQL Server (CloudLabs), BCrypt password hashing, cookie authentication with role-based access.

[![.NET CI](../../actions/workflows/dotnet.yml/badge.svg)](../../actions/workflows/dotnet.yml)

## Requirements
- .NET 10 SDK
- Access to the RACEDAY SQL Server database on CloudLabs (the schema already exists - the API does not create it)
- Git

## 1. Configure the database connection (never commit it)
`appsettings.json` deliberately contains an **empty** connection string. Supply the real one in one of two ways.

**Option A - environment variable (PowerShell, current terminal only):**
```powershell
$env:ConnectionStrings__DefaultConnection = "Server=YOUR_SERVER;Database=RACEDAY;User Id=YOUR_USER;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=True"
```

**Option B - .NET user secrets (stored outside the repo, per developer):**
```bash
cd RaceDay.Api
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=YOUR_SERVER;Database=RACEDAY;User Id=YOUR_USER;Password=YOUR_PASSWORD;Encrypt=True;TrustServerCertificate=True"
```
Use the server name, database name and login given in your CloudLabs instructions.
If the connection string is missing, the first database request fails with a message saying so.

## 2. Run the API
```bash
dotnet restore RaceDay.Api.Tests/RaceDay.Api.Tests.csproj
dotnet run --project RaceDay.Api/RaceDay.Api.csproj --launch-profile https
```
Open **https://localhost:7205/swagger** (Swagger is only enabled in the Development environment, which the launch profile sets).
If the browser warns about the development certificate, run `dotnet dev-certs https --trust` once.

## 3. Authentication and roles
Authentication uses an **HTTP-only cookie** (not JWT). Passwords are stored only as BCrypt hashes and are never returned by the API.

| Role | Can do |
|------|--------|
| anonymous | read events, categories and results; register; log in |
| Runner | everything anonymous can + enrol **themselves**, see/update their own enrolments and profile |
| Organiser | Runner rights + create/update events and categories **they own** |
| Admin | everything, including users, results and all DELETE operations |

**How to log in with Swagger**
1. `POST /Auth/register` - creates a **Runner** (role cannot be chosen at registration).
2. `POST /Auth/login` - Swagger/browser stores the cookie automatically.
3. `GET /Auth/me` - confirms who you are logged in as. `POST /Auth/logout` ends the session.

**Creating the first Admin:** register normally, then promote that user directly in SQL Server:
```sql
UPDATE Users SET Role = 'Admin' WHERE Email = 'admin@example.com';
```
Log out and in again so the new role is in your cookie. After that, an Admin can create Organisers/Admins with `POST /Users`.

Unauthenticated calls return **401**, calls without the required role return **403** (no redirects).

## 4. Endpoints
| Resource | Endpoints | Access |
|----------|-----------|--------|
| Auth | `POST /Auth/register`, `POST /Auth/login`, `GET /Auth/me`, `POST /Auth/logout` | register/login public; me/logout need login |
| Users | `GET /Users`, `GET /Users/{id}`, `POST /Users`, `PUT /Users/{id}`, `DELETE /Users/{id}` | list/create/delete Admin; get/update self or Admin (role change Admin only) |
| Events | `GET /Events`, `GET /Events/{id}` | public |
| | `POST /Events`, `PUT /Events/{id}` | Organiser (own events) / Admin |
| | `DELETE /Events/{id}` | Admin |
| Categories | `GET /Categories`, `GET /Categories/{id}` | public |
| | `POST /Categories`, `PUT /Categories/{id}` | Organiser (own events) / Admin |
| | `DELETE /Categories/{id}` | Admin |
| Enrolments | `GET /Enrolments`, `GET /Enrolments/{id}` | logged in (Runners see only their own) |
| | `POST /Enrolments`, `PUT /Enrolments/{id}` | logged in (self) / Admin (anyone) |
| | `DELETE /Enrolments/{id}` | Admin |
| Results | `GET /Results`, `GET /Results/{id}` | public |
| | `POST /Results`, `PUT /Results/{id}`, `DELETE /Results/{id}` | Admin |

Swagger shows the full request/response schemas.

### Status codes used
| Code | Meaning |
|------|---------|
| 200 / 201 / 204 | success / created / success with no body |
| 400 | invalid input (validation) or a reference to something that does not exist (bad foreign key, unknown event type) |
| 401 / 403 | not logged in / logged in without permission |
| 404 | record not found |
| 409 | duplicate (email, category name per event, enrolment, result) or record still referenced (e.g. deleting an event that has categories) |

### Request examples
```jsonc
// POST /Auth/register
{ "firstName": "Thandi", "lastName": "Nkosi", "email": "thandi@example.com", "password": "Secret123!" }

// POST /Events   (eventType must match an existing EventTypes.EventTypeName)
{ "eventName": "Cape Town 10K", "eventType": "Road Run", "eventDate": "2026-12-01", "location": "Green Point", "province": "Western Cape" }

// POST /Categories
{ "eventId": 1, "name": "10km", "distanceKm": 10, "maxParticipant": 500, "entryFee": 150 }

// POST /Enrolments  (userId optional - defaults to you)
{ "categoryId": 1 }

// POST /Results  (Admin)
{ "enrolment": 1, "finishTime": "00:45:30", "position": 1 }
```

## 5. Automated tests
```bash
dotnet test RaceDay.Api.Tests/RaceDay.Api.Tests.csproj
```
The tests start the real API in memory (real routing, cookie login, roles and validation) with an EF Core **InMemory**
database - no SQL Server or secrets needed. They cover registration/hashing, login/logout, 401 vs 403 vs 404,
invalid input, duplicates (409), referenced-record deletes (409) and that data is actually saved/removed, for every controller.

Limitation: InMemory does not enforce SQL foreign keys or unique indexes. Those are checked by the controllers (tested)
and verified against the real database with the manual steps below.

## 6. Verifying SQL Server
1. Follow `docs/TEST_PLAN.md` in Swagger and screenshot each result.
2. Run `docs/verify-database.sql` in SSMS/Azure Data Studio to confirm foreign keys, unique constraints, hashed passwords
   and that the rows created through the API are present. Restart the API and repeat a `GET` to prove persistence.

## 7. GitHub and CI
`.github/workflows/dotnet.yml` restores, builds and tests on every push and pull request to `main`/`master`.
CI needs no database or secrets. Check the **Actions** tab for a green run after each push.

## Security notes
- Connection strings/credentials live only in environment variables or user secrets - never in the repository.
- Passwords are hashed with BCrypt; `PasswordHash` is never part of any response.
- Auth cookie: HTTP-only, SameSite=Lax, Secure outside Development, 1-hour sliding expiry. Role changes apply at next login.
- Roles must be stored exactly as `Admin`, `Organiser` or `Runner` (see `Security/AppRoles.cs`).
