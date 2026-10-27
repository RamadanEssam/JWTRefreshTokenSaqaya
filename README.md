# JWT and Refresh Token API — .NET 6 Practice

A learning API showing registration, JWT access tokens, refresh-token rotation, and revocation using ASP.NET Core Identity and SQL Server.

**Status:** historical educational sample, not a production authentication service. Do not expose it publicly without a separate security and dependency review.

## Implemented flow

```text
Register or sign in → JWT access token + refresh-token cookie
→ Call a protected endpoint with the JWT
→ Refresh using the cookie → previous refresh token revoked
→ Revoke a refresh token when it is no longer needed
```

The repository contains `AuthController`, `AuthService`, Identity-backed `ApplicationDbContext`, JWT validation settings, and EF Core migrations. The implementation uses a 10-day refresh-token lifetime; the access-token lifetime comes from configuration.

## Stack and structure

- ASP.NET Core / `net6.0`.
- ASP.NET Core Identity and EF Core SQL Server 6.0.1.
- JWT bearer authentication and System.IdentityModel.Tokens.Jwt 6.15.1.
- Swagger for development.

```text
JWTRefreshTokenSaqaya.sln
└── JWTRefreshTokenInDotNet6/
    ├── Controllers/  # Auth endpoints and an authorized endpoint
    ├── Services/     # Token creation, refresh, revocation, role assignment
    ├── Models/       # Identity user, request/response models, DbContext
    ├── Settings/     # JWT settings shape
    └── Migrations/   # Identity and refresh-token schema
```

## Restore and build

Requires an SDK capable of building .NET 6, the ASP.NET Core 6 runtime for execution, and a local SQL Server instance. .NET 6 is out of support; do not interpret a successful build as a production-readiness check.

```powershell
dotnet restore JWTRefreshTokenSaqaya.sln --source https://api.nuget.org/v3/index.json
dotnet build JWTRefreshTokenSaqaya.sln --no-restore
```

## Safe local configuration

The application reads configuration through the standard ASP.NET Core builder. Override the checked-in settings locally instead of reusing committed sample signing material:

```powershell
$env:ConnectionStrings__DefaultConnection = 'Server=localhost;Database=JwtRefreshDemo;Trusted_Connection=True;TrustServerCertificate=True'
$env:JWT__Issuer = 'JwtRefreshDemo'
$env:JWT__Audience = 'JwtRefreshDemoClient'
$env:JWT__DurationInMinutes = '15'

# Generate a fresh local signing secret; do not print or commit it.
$signingBytes = New-Object byte[] 64
$random = [System.Security.Cryptography.RandomNumberGenerator]::Create()
$random.GetBytes($signingBytes)
$random.Dispose()
$env:JWT__Key = [Convert]::ToBase64String($signingBytes)
```

Environment variables apply to this terminal and child processes. Restarting with a different signing secret invalidates access tokens signed with the old one. These are development instructions, not a production secret-management design.

## Initialize a development database

Migrations are committed, including a migration that creates the `User` role. If compatible EF tooling is not installed, prepare a local tool on your machine:

```powershell
dotnet new tool-manifest
dotnet tool install dotnet-ef --version 6.0.1 --add-source https://api.nuget.org/v3/index.json
dotnet ef database update --project JWTRefreshTokenInDotNet6/JWTRefreshTokenSaqaya.csproj
```

Run from the same configured terminal:

```powershell
dotnet run --project JWTRefreshTokenInDotNet6/JWTRefreshTokenSaqaya.csproj --launch-profile JWTRefreshTokenInDotNet6
```

The checked-in launch profile exposes `https://localhost:7029` and `http://localhost:5029`. Use `/swagger` on the HTTPS endpoint in development; configure/trust the local development certificate as needed.

## API walkthrough

| Endpoint | Purpose |
|---|---|
| `POST /api/Auth/register` | Create a user and return authentication data |
| `POST /api/Auth/token` | Sign in with email and password |
| `GET /api/Auth/refreshToken` | Read the refresh-token cookie and rotate the token |
| `POST /api/Auth/revokeToken` | Revoke a token provided in the body or cookie |
| `POST /api/Auth/addRole` | Assign a role — currently lacks an authorization attribute |
| `GET /api/Secured` | Example endpoint protected by `[Authorize]` |

A registration body has `firstName`, `lastName`, `username`, `email`, and `password`. A sign-in body has `email` and `password`. Use only synthetic local test accounts. Use a client that preserves cookies for the refresh flow and send the returned access token as `Authorization: Bearer <access-token>` to the protected endpoint.

## Important limitations found in the code

- `addRole` is not protected by an explicit authorization policy; this is not safe for public deployment.
- Refresh-token cookies are HttpOnly, but do not explicitly set Secure/SameSite protections in the current helper.
- Refresh tokens are stored as token strings rather than hashes.
- Registration request validation is limited, and nullable-reference warnings remain.
- Committed sample settings must not be reused as real secrets. If any committed credential has ever been used outside a local exercise, rotate it separately.
- Framework and dependencies need modernization. Restore reports a known vulnerability for System.IdentityModel.Tokens.Jwt 6.15.1: [GHSA-59j7-ghrg-fj52](https://github.com/advisories/GHSA-59j7-ghrg-fj52).
- No hosted demo, automated security validation, or production deployment is claimed.
- No root project license was found; this documentation does not add one.

## Verification

On 2026-10-27, restore and build succeeded using .NET SDK 10.0.402 with **27 warnings and 0 errors** on the reviewed checkout. Warnings include the unsupported target, package advisory, nullable references, and obsolete RNG API. Database migrations and HTTP flows were not executed during this review. Application behavior and dependencies were left unchanged.
