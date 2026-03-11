# 핸즈온랩: 100줄로 만드는 MCP 서버

> **.NET 10 파일 기반 앱 + SQLite + MCP — 빈 폴더에서 AI 도구 서버까지**

---

## 모듈 개요

| 항목 | 내용 |
|------|------|
| 소요 시간 | 30분 |
| 난이도 | 초중급 |
| 대상 | C# 기초 문법을 아는 개발자 |
| 목표 | .NET 10 파일 기반 앱으로 MCP 서버를 단일 C# 파일에 구현 |
| 준비물 | .NET 10 SDK, VS Code, macOS/Windows/Linux |

### 학습 목표

이 핸즈온랩을 완료하면 다음을 할 수 있습니다.

1. .NET 10의 파일 기반 앱이 무엇인지 설명하고, 프로젝트 파일 없이 C# 파일 하나로 애플리케이션을 실행할 수 있다.
2. SQLite 데이터베이스를 생성하고 초기 데이터를 투입할 수 있다.
3. MCP(Model Context Protocol) 서버의 기본 구조를 이해하고, AI 클라이언트가 호출할 수 있는 도구(Tool)를 등록할 수 있다.
4. 위 세 가지를 조합하여 100줄 내외의 단일 파일로 동작하는 MCP 서버를 완성할 수 있다.

---

## 사전 지식: 왜 100줄이면 충분한가?

대부분의 개발자는 "MCP 서버를 100줄로 만든다"는 문제를 보면 불가능하다고 판단합니다. `.csproj` 파일을 만들고, 프로젝트 구조를 잡고, NuGet 패키지를 구성하는 것만으로도 상당한 분량이 필요하다고 생각하기 때문입니다.

그러나 **.NET 10의 파일 기반 앱(File-based App)** 을 알고 있다면 이야기가 달라집니다. 파일 기반 앱은 다음과 같은 특성을 갖습니다.

- **프로젝트 파일이 필요 없습니다.** `.csproj` 없이 `.cs` 파일 하나로 실행됩니다. .NET SDK가 소스 파일의 지시문을 기반으로 프로젝트 구성을 자동 생성합니다.
- **NuGet 패키지를 파일 내에서 선언합니다.** `#:package` 지시문으로 의존성을 직접 지정합니다.
- **실행 방법이 간결합니다.** `dotnet run app.cs`, `dotnet app.cs`, 또는 Unix에서는 shebang을 통해 `./app.cs`로 직접 실행할 수 있습니다.

이 세 가지 특성 덕분에 보일러플레이트가 극적으로 줄어들고, 핵심 로직에만 집중할 수 있습니다.

### 파일 기반 앱이 지원하는 지시문(Directive)

파일 기반 앱은 `#:` 접두사가 붙는 지시문을 소스 파일 상단에 배치하여 빌드와 실행을 구성합니다. 이 랩에서 사용하는 지시문을 포함하여 주요 지시문은 다음과 같습니다.

| 지시문 | 역할 | 예시 |
|--------|------|------|
| `#:package` | NuGet 패키지 참조 추가 | `#:package Newtonsoft.Json@13.0.3` |
| `#:sdk` | 사용할 SDK 지정 (기본값: `Microsoft.NET.Sdk`) | `#:sdk Microsoft.NET.Sdk.Web` |
| `#:property` | MSBuild 속성 설정 | `#:property PublishAot=false` |
| `#:project` | 다른 프로젝트 파일 참조 | `#:project ../Shared/Shared.csproj` |

패키지 버전을 지정하지 않으면 Central Package Management가 필요합니다. 그렇지 않은 환경에서는 `@*`(최신 버전) 또는 명시적 버전 번호를 지정해야 합니다.

---

## Step 0: 환경 준비 (사전 작업)

### .NET 10 SDK 설치 확인

```bash
dotnet --version
```

출력이 `10.0.100` 이상이어야 합니다. 그렇지 않다면 [dotnet.microsoft.com](https://dotnet.microsoft.com/download/dotnet/10.0)에서 .NET 10 SDK를 설치하십시오.

### VS Code 확장 설치

- **C# Dev Kit** 확장이 설치되어 있어야 합니다.

### 작업 폴더 생성

```bash
mkdir mcp-lab && cd mcp-lab
```

> 📌 **체크포인트**: `dotnet --version`이 `10.0.100` 이상 출력되고, 빈 폴더 안에 있으면 준비 완료입니다.

---

## Step 1: SQLite 데이터베이스 준비

MCP 서버가 질의할 대상 데이터를 먼저 준비합니다. 이 랩에서는 간단한 도서 목록 데이터베이스를 사용합니다.

### 1-1. SQLite CLI로 데이터베이스 생성

```bash
sqlite3 books.db <<'EOF'
CREATE TABLE IF NOT EXISTS books (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    year INTEGER NOT NULL,
    genre TEXT NOT NULL
);

INSERT INTO books (title, author, year, genre) VALUES
    ('클린 코드', 'Robert C. Martin', 2008, '소프트웨어 공학'),
    ('리팩터링', 'Martin Fowler', 2018, '소프트웨어 공학'),
    ('도메인 주도 설계', 'Eric Evans', 2003, '소프트웨어 공학'),
    ('디자인 패턴', 'GoF', 1994, '소프트웨어 공학'),
    ('실용주의 프로그래머', 'Hunt & Thomas', 1999, '소프트웨어 공학'),
    ('코딩 인터뷰 완전 분석', 'Gayle McDowell', 2015, '취업'),
    ('가상 면접 사례로 배우는 대규모 시스템 설계', 'Alex Xu', 2020, '시스템 설계'),
    ('데이터 중심 애플리케이션 설계', 'Martin Kleppmann', 2017, '데이터 공학'),
    ('HTTP 완벽 가이드', 'David Gourley', 2002, '웹'),
    ('운영체제 아주 쉬운 세 가지 이야기', 'Remzi Arpaci-Dusseau', 2018, '운영체제');
EOF
```

### 1-2. 데이터 확인

```bash
sqlite3 books.db "SELECT * FROM books;"
```

10건의 도서 데이터가 출력되면 성공입니다.

> 📌 **체크포인트**: `books.db` 파일이 작업 폴더에 존재하고, 10건의 레코드가 조회되어야 합니다.

---

## Step 2: C# 파일 작성 — 파일 기반 앱의 골격

### 2-1. 파일 생성

`app.cs` 파일을 생성하고, 파일 기반 앱의 골격을 작성합니다.

```csharp
#!/usr/bin/env dotnet
#:package Microsoft.Data.Sqlite@9.*
#:package ModelContextProtocol@0.*
#:package Microsoft.Extensions.Hosting@10.*
```

위 네 줄이 파일 기반 앱의 핵심입니다.

- **첫 번째 줄 (Shebang)**: Unix 계열에서 `./app.cs`로 직접 실행할 수 있게 합니다. `#!/usr/bin/env dotnet`은 PATH에서 dotnet 경로를 자동으로 찾아줍니다. Windows에서는 무시되지만, 파일 기반 앱에서는 모든 플랫폼에서 이 지시문을 허용합니다.
- **`#:package` 지시문**: NuGet 패키지 의존성을 파일 내에서 직접 선언합니다. `@9.*`처럼 와일드카드 버전을 사용하면 해당 메이저 버전의 최신 패키지를 자동으로 가져옵니다.

### 2-2. 필요한 네임스페이스 선언

```csharp
using Microsoft.Data.Sqlite;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ModelContextProtocol.Server;
using System.ComponentModel;
```

> 📌 **체크포인트**: 여기까지 작성한 뒤 `dotnet run app.cs`를 실행하여 패키지가 정상 복원되는지 확인합니다. (아직 본문 코드가 없으므로 빈 실행이 됩니다.)

---

## Step 3: MCP 서버 호스트 구성

### 3-1. Generic Host 기반 MCP 서버 설정

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();

await builder.Build().RunAsync();
```

이 코드는 다음을 수행합니다.

- `AddMcpServer()`: MCP 서버를 DI 컨테이너에 등록합니다.
- `WithStdioServerTransport()`: 표준 입출력(stdio)을 전송 계층으로 사용합니다. Claude Desktop, VS Code GitHub Copilot 등 대부분의 MCP 클라이언트가 이 방식을 지원합니다. 원격 배포가 필요한 경우 `WithHttpTransport()`로 전환할 수 있으며, 이때는 `#:sdk Microsoft.NET.Sdk.Web`과 `ModelContextProtocol.AspNetCore` 패키지가 필요합니다.
- `WithToolsFromAssembly()`: 현재 어셈블리에서 `[McpServerToolType]`이 붙은 클래스의 `[McpServerTool]` 메서드를 자동 검색하여 도구로 등록합니다.

> 📌 **체크포인트**: `dotnet run app.cs`를 실행하면 프로세스가 대기 상태로 진입합니다. stdio 기반이므로 터미널에서 입력을 기다리는 것이 정상입니다. `Ctrl+C`로 종료합니다.

---

## Step 4: MCP 도구(Tool) 구현

MCP 서버의 핵심은 **도구(Tool)** 입니다. AI 클라이언트는 이 도구를 호출하여 외부 리소스와 상호작용합니다. 도구 클래스에는 `[McpServerToolType]` 어트리뷰트를, 개별 메서드에는 `[McpServerTool]` 어트리뷰트를 붙이고, `[Description]`으로 AI가 도구의 용도와 매개변수를 이해할 수 있도록 설명을 제공합니다. 이 랩에서는 세 가지 도구를 구현합니다.

### 4-1. 도구 클래스 선언

```csharp
[McpServerToolType]
public static class BookTools
{
    private const string DbPath = "books.db";
```

### 4-2. 도구 1 — 전체 도서 목록 조회

```csharp
    [McpServerTool, Description("등록된 모든 도서 목록을 조회합니다.")]
    public static string ListBooks()
    {
        using var conn = new SqliteConnection($"Data Source={DbPath}");
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT id, title, author, year, genre FROM books ORDER BY id";

        using var reader = cmd.ExecuteReader();
        var results = new System.Text.StringBuilder();
        while (reader.Read())
        {
            results.AppendLine(
                $"[{reader.GetInt32(0)}] {reader.GetString(1)} — {reader.GetString(2)} ({reader.GetInt32(3)}, {reader.GetString(4)})");
        }

        return results.Length > 0 ? results.ToString() : "등록된 도서가 없습니다.";
    }
```

### 4-3. 도구 2 — 키워드 검색

```csharp
    [McpServerTool, Description("제목 또는 저자명으로 도서를 검색합니다.")]
    public static string SearchBooks(
        [Description("검색할 키워드 (제목 또는 저자명)")] string keyword)
    {
        using var conn = new SqliteConnection($"Data Source={DbPath}");
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText = """
            SELECT id, title, author, year, genre FROM books
            WHERE title LIKE @kw OR author LIKE @kw
            ORDER BY id
            """;
        cmd.Parameters.AddWithValue("@kw", $"%{keyword}%");

        using var reader = cmd.ExecuteReader();
        var results = new System.Text.StringBuilder();
        while (reader.Read())
        {
            results.AppendLine(
                $"[{reader.GetInt32(0)}] {reader.GetString(1)} — {reader.GetString(2)} ({reader.GetInt32(3)}, {reader.GetString(4)})");
        }

        return results.Length > 0 ? results.ToString() : $"'{keyword}'에 해당하는 도서를 찾을 수 없습니다.";
    }
```

### 4-4. 도구 3 — 장르별 조회

```csharp
    [McpServerTool, Description("특정 장르에 해당하는 도서 목록을 조회합니다.")]
    public static string GetBooksByGenre(
        [Description("조회할 장르 (예: 소프트웨어 공학, 시스템 설계, 웹)")] string genre)
    {
        using var conn = new SqliteConnection($"Data Source={DbPath}");
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText = """
            SELECT id, title, author, year, genre FROM books
            WHERE genre = @genre
            ORDER BY year
            """;
        cmd.Parameters.AddWithValue("@genre", genre);

        using var reader = cmd.ExecuteReader();
        var results = new System.Text.StringBuilder();
        while (reader.Read())
        {
            results.AppendLine(
                $"[{reader.GetInt32(0)}] {reader.GetString(1)} — {reader.GetString(2)} ({reader.GetInt32(3)})");
        }

        return results.Length > 0 ? results.ToString() : $"'{genre}' 장르의 도서를 찾을 수 없습니다.";
    }
}
```

> 📌 **체크포인트**: 파일 전체를 저장한 뒤 `dotnet run app.cs`가 오류 없이 실행되는지 확인합니다.

---

## Step 5: 완성된 코드 전문

아래는 `app.cs`의 전체 코드입니다. 공백 포함 약 90줄입니다.

```csharp
#!/usr/bin/env dotnet
#:package Microsoft.Data.Sqlite@9.*
#:package ModelContextProtocol@0.*
#:package Microsoft.Extensions.Hosting@10.*

using Microsoft.Data.Sqlite;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ModelContextProtocol.Server;
using System.ComponentModel;

var builder = Host.CreateApplicationBuilder(args);

builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();

await builder.Build().RunAsync();

// ── MCP Tools ──────────────────────────────────────────

[McpServerToolType]
public static class BookTools
{
    private const string DbPath = "books.db";

    [McpServerTool, Description("등록된 모든 도서 목록을 조회합니다.")]
    public static string ListBooks()
    {
        using var conn = new SqliteConnection($"Data Source={DbPath}");
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT id, title, author, year, genre FROM books ORDER BY id";

        using var reader = cmd.ExecuteReader();
        var results = new System.Text.StringBuilder();
        while (reader.Read())
        {
            results.AppendLine(
                $"[{reader.GetInt32(0)}] {reader.GetString(1)} — {reader.GetString(2)} ({reader.GetInt32(3)}, {reader.GetString(4)})");
        }

        return results.Length > 0 ? results.ToString() : "등록된 도서가 없습니다.";
    }

    [McpServerTool, Description("제목 또는 저자명으로 도서를 검색합니다.")]
    public static string SearchBooks(
        [Description("검색할 키워드 (제목 또는 저자명)")] string keyword)
    {
        using var conn = new SqliteConnection($"Data Source={DbPath}");
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText = """
            SELECT id, title, author, year, genre FROM books
            WHERE title LIKE @kw OR author LIKE @kw
            ORDER BY id
            """;
        cmd.Parameters.AddWithValue("@kw", $"%{keyword}%");

        using var reader = cmd.ExecuteReader();
        var results = new System.Text.StringBuilder();
        while (reader.Read())
        {
            results.AppendLine(
                $"[{reader.GetInt32(0)}] {reader.GetString(1)} — {reader.GetString(2)} ({reader.GetInt32(3)}, {reader.GetString(4)})");
        }

        return results.Length > 0 ? results.ToString() : $"'{keyword}'에 해당하는 도서를 찾을 수 없습니다.";
    }

    [McpServerTool, Description("특정 장르에 해당하는 도서 목록을 조회합니다.")]
    public static string GetBooksByGenre(
        [Description("조회할 장르 (예: 소프트웨어 공학, 시스템 설계, 웹)")] string genre)
    {
        using var conn = new SqliteConnection($"Data Source={DbPath}");
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText = """
            SELECT id, title, author, year, genre FROM books
            WHERE genre = @genre
            ORDER BY year
            """;
        cmd.Parameters.AddWithValue("@genre", genre);

        using var reader = cmd.ExecuteReader();
        var results = new System.Text.StringBuilder();
        while (reader.Read())
        {
            results.AppendLine(
                $"[{reader.GetInt32(0)}] {reader.GetString(1)} — {reader.GetString(2)} ({reader.GetInt32(3)})");
        }

        return results.Length > 0 ? results.ToString() : $"'{genre}' 장르의 도서를 찾을 수 없습니다.";
    }
}
```

---

## Step 6: MCP 클라이언트에서 연결 확인

구현한 MCP 서버는 stdio 전송을 사용하므로, MCP 클라이언트가 프로세스를 직접 실행하는 방식으로 연결합니다. 아래에서 주요 클라이언트별 설정 방법을 안내합니다.

### 6-1. VS Code (GitHub Copilot) 설정

VS Code에서 GitHub Copilot의 Agent 모드를 통해 MCP 서버에 연결할 수 있습니다. 프로젝트 루트에 `.vscode/mcp.json` 파일을 생성합니다.

```json
{
  "servers": {
    "book-db": {
      "type": "stdio",
      "command": "dotnet",
      "args": ["run", "app.cs"]
    }
  }
}
```

VS Code에서 GitHub Copilot Chat을 열고 **Agent 모드**로 전환한 뒤, **도구 선택(Select Tools)** 아이콘에서 `book-db` 서버가 표시되는지 확인합니다. 도구 목록에 `ListBooks`, `SearchBooks`, `GetBooksByGenre`가 나타나면 정상입니다.

### 6-2. Claude Desktop 설정

Claude Desktop을 사용하는 경우, `claude_desktop_config.json` 파일에 다음을 추가합니다.

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "book-db": {
      "command": "dotnet",
      "args": ["run", "app.cs"],
      "cwd": "/path/to/mcp-lab"
    }
  }
}
```

`cwd`를 실제 작업 폴더의 절대 경로로 수정하십시오.

### 6-3. 테스트

클라이언트를 재시작한 뒤, 다음과 같이 자연어로 질의합니다.

- "등록된 도서 목록을 보여주세요."
- "Martin이 쓴 책을 찾아주세요."
- "소프트웨어 공학 장르의 책은 뭐가 있나요?"

AI가 MCP 도구를 호출하여 SQLite 데이터베이스에서 결과를 가져오는 것을 확인할 수 있습니다.

> 📌 **체크포인트**: 클라이언트의 도구 목록에 `ListBooks`, `SearchBooks`, `GetBooksByGenre` 세 가지가 표시되고, 자연어 질의에 대해 정확한 결과가 반환되면 완료입니다.

---

## 정리: 핵심 포인트

### .NET 10 파일 기반 앱이 가능하게 한 것

| 기존 방식 | 파일 기반 앱 |
|-----------|-------------|
| `.csproj` + `Program.cs` + NuGet 구성 | `.cs` 파일 1개 |
| `dotnet new` → `dotnet add package` → `dotnet run` | `dotnet run app.cs` |
| 프로젝트 단위 빌드 | 파일 단위 실행 |
| 별도 publish 설정 필요 | Native AOT 기본 활성화 |

### 파일 기반 앱의 주요 CLI 명령어

| 명령어 | 설명 |
|--------|------|
| `dotnet run app.cs` | 파일 기반 앱 실행 |
| `dotnet app.cs` | 축약형 실행 (동일) |
| `dotnet build app.cs` | 빌드만 수행 |
| `dotnet publish app.cs` | Native AOT 바이너리 생성 |
| `dotnet clean app.cs` | 빌드 산출물 정리 |
| `dotnet project convert app.cs` | 전통적인 프로젝트 구조로 변환 |

### MCP 서버의 최소 구성 요소

| 구성 요소 | 역할 | 코드에서의 위치 |
|-----------|------|----------------|
| Host | 서버 수명 주기 관리 | `Host.CreateApplicationBuilder` |
| Transport | 클라이언트와의 통신 방식 | `WithStdioServerTransport()` |
| Tool Type | 도구 메서드를 포함하는 클래스 | `[McpServerToolType]` 어트리뷰트 |
| Tool | AI가 호출하는 기능 단위 | `[McpServerTool]` 어트리뷰트 |
| Description | AI가 도구의 용도를 이해하기 위한 설명 | `[Description]` 어트리뷰트 |

---

## 도전 과제 (선택)

시간이 남는 참가자를 위한 확장 과제입니다.

### 도전 1: 도서 추가 도구 구현

`AddBook(string title, string author, int year, string genre)` 도구를 추가하여 AI가 데이터베이스에 새 도서를 등록할 수 있게 하십시오.

### 도전 2: 통계 도구 구현

`GetStats()` 도구를 추가하여 장르별 도서 수, 전체 도서 수, 가장 오래된/최신 도서 등의 통계를 반환하십시오.

### 도전 3: 자신만의 도메인 적용

도서 대신 자신이 관심 있는 도메인(영화, 게임, 레시피 등)의 데이터로 교체하여 나만의 MCP 서버를 완성하십시오.

### 도전 4: NuGet 패키지로 배포

.NET 10은 MCP 서버를 NuGet 패키지로 배포하는 워크플로를 공식 지원합니다. `dotnet project convert app.cs`로 프로젝트 구조로 변환한 뒤, `dotnet new mcpserver` 템플릿의 `server.json` 형식을 참고하여 NuGet.org에 배포할 수 있는 형태로 패키징해 보십시오. 배포된 MCP 서버는 `dnx` 명령어를 통해 설치 없이 바로 실행할 수 있습니다.

---

## 더 알아보기: 프로젝트 기반 MCP 서버 템플릿

이 핸즈온랩에서는 파일 기반 앱의 간결함을 체험하는 데 집중했지만, 실무에서 더 복잡한 MCP 서버를 구축할 때는 공식 프로젝트 템플릿을 사용하는 것이 효율적입니다.

```bash
dotnet new install Microsoft.McpServer.ProjectTemplates
dotnet new mcpserver -n MyMcpServer
```

이 템플릿은 stdio와 HTTP 전송 중 선택할 수 있고, Native AOT 및 Self-contained 배포 옵션을 제공하며, NuGet 배포를 위한 `server.json` 파일이 포함됩니다.

---

## 참고 자료

- [.NET 10 파일 기반 앱 공식 문서](https://learn.microsoft.com/dotnet/core/sdk/file-based-apps)
- [튜토리얼: 파일 기반 C# 프로그램 만들기](https://learn.microsoft.com/dotnet/csharp/fundamentals/tutorials/file-based-programs)
- [.NET AI 및 MCP 시작하기](https://learn.microsoft.com/dotnet/ai/get-started-mcp)
- [퀵스타트: C#으로 MCP 서버 만들기](https://learn.microsoft.com/dotnet/ai/quickstarts/build-mcp-server)
- [NuGet의 MCP 서버 패키지](https://learn.microsoft.com/nuget/concepts/nuget-mcp)
- [Model Context Protocol 공식 사이트](https://modelcontextprotocol.io)
- [MCP C# SDK (GitHub)](https://github.com/modelcontextprotocol/csharp-sdk)
- [ModelContextProtocol NuGet 패키지](https://www.nuget.org/packages/ModelContextProtocol)
- [Microsoft.Data.Sqlite 문서](https://learn.microsoft.com/dotnet/standard/data/sqlite/)

---

## 부록: 바이브 코딩을 위한 MCP 서버 설정

이 핸즈온랩에서는 MCP 서버를 직접 만드는 것에 집중했습니다. 하지만 실무에서는 **이미 만들어진 MCP 서버를 AI 코딩 에이전트에 연결하여 바이브 코딩(Vibe Coding)의 정확도를 높이는 것**이 훨씬 빈번한 사용 사례입니다.

AI 코딩 에이전트는 학습 데이터에 기반하여 API를 추측하기 때문에, 실제로 존재하지 않는 메서드를 생성하거나 매개변수 시그니처를 틀리는 환각(hallucination)이 발생합니다. MCP 서버를 연결하면 에이전트가 **실제 문서와 어셈블리 메타데이터를 직접 조회**할 수 있으므로, 이러한 오류를 크게 줄일 수 있습니다.

여기서는 .NET 바이브 코딩에 특히 유용한 두 가지 MCP 서버를 소개합니다.

### A-1. Microsoft Learn MCP 서버

Microsoft Learn MCP 서버는 Microsoft 공식 문서를 AI 에이전트가 직접 검색하고 참조할 수 있게 해주는 원격(Remote) MCP 서버입니다. .NET, Azure, C#, ASP.NET Core, Entity Framework 등 Microsoft 기술 스택 전반에 대한 최신 공식 문서에 접근할 수 있습니다.

제공하는 주요 도구는 다음과 같습니다.

| 도구 | 용도 |
|------|------|
| `microsoft_docs_search` | Microsoft Learn 문서 키워드 검색 |
| `microsoft_docs_fetch` | 특정 문서 페이지의 전체 내용 조회 |
| `microsoft_code_sample_search` | 공식 코드 샘플 검색 |

### A-2. HandMirrorMcp

HandMirrorMcp는 .NET 어셈블리와 NuGet 패키지를 직접 검사(inspect)하는 로컬(Local) MCP 서버입니다. "Mirror, mirror, on the wall — show me what this API really calls"라는 태그라인처럼, AI 에이전트가 코드를 작성하기 전에 실제 API의 타입, 메서드 시그니처, 네임스페이스를 확인할 수 있게 해줍니다.

제공하는 주요 도구 카테고리는 다음과 같습니다.

| 카테고리 | 주요 도구 | 용도 |
|----------|-----------|------|
| 어셈블리 검사 | `inspect_assembly`, `get_type_info`, `list_namespaces` | 어셈블리의 공개 타입, 멤버, XML 문서 분석 |
| NuGet 패키지 탐색 | `search_nuget_packages`, `inspect_nuget_package`, `inspect_nuget_package_type` | 패키지 검색, 어셈블리 분석, 타입 상세 조회 |
| 네이티브 인터롭 | `inspect_native_dependencies` | P/Invoke, COM 타입 의존성 분석 |
| 프로젝트 분석 | `analyze_csproj`, `analyze_solution`, `explain_build_error` | 프로젝트 파일 분석, 빌드 오류 진단 |
| 파일 기반 앱 분석 | `analyze_file_based_app` | .NET 10 파일 기반 앱 분석 |

### A-3. 클라이언트별 설정 방법

#### VS Code (GitHub Copilot)

프로젝트 루트의 `.vscode/mcp.json` 파일에 다음을 추가합니다.

```json
{
  "servers": {
    "microsoft-learn": {
      "type": "http",
      "url": "https://learn.microsoft.com/api/mcp"
    },
    "HandMirrorMcp": {
      "type": "stdio",
      "command": "dnx",
      "args": ["HandMirrorMcp@0.1.1", "--yes"]
    }
  }
}
```

설정 후 GitHub Copilot Chat에서 Agent 모드로 전환하고, 도구 선택(Select Tools) 아이콘에서 두 서버가 모두 표시되는지 확인합니다. VS Code Command Palette(`Ctrl+Shift+P` / `Cmd+Shift+P`)에서 `MCP: Add Server`를 통해 UI로 추가할 수도 있습니다.

> **참고**: `dnx`는 .NET 10 SDK에 포함된 명령어로, NuGet에 게시된 .NET 도구를 사전 설치 없이 바로 실행합니다. `--yes` 플래그는 최초 실행 시 확인 프롬프트를 건너뜁니다.

#### Cursor

프로젝트 루트의 `.cursor/mcp.json` 파일에 다음을 추가합니다.

```json
{
  "mcpServers": {
    "microsoft-learn": {
      "url": "https://learn.microsoft.com/api/mcp"
    },
    "HandMirrorMcp": {
      "command": "dnx",
      "args": ["HandMirrorMcp@0.1.1", "--yes"]
    }
  }
}
```

Cursor Settings → MCP 탭에서 서버가 활성화되었는지 확인합니다.

#### Claude Desktop

`claude_desktop_config.json` 파일에 다음을 추가합니다.

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "HandMirrorMcp": {
      "command": "dnx",
      "args": ["HandMirrorMcp@0.1.1", "--yes"]
    }
  }
}
```

> **참고**: Claude Desktop은 현재 stdio 전송 방식의 로컬 MCP 서버를 지원합니다. Microsoft Learn MCP 서버는 원격(HTTP) 방식이므로 Claude Desktop에서는 직접 연결할 수 없지만, Claude Code에서는 플러그인 방식으로 사용할 수 있습니다.

#### Claude Code

Claude Code에서는 플러그인 방식으로 Microsoft Learn MCP 서버를 설치할 수 있습니다.

```shell
/plugin marketplace add microsoftdocs/mcp
/plugin install microsoft-docs@microsoft-docs-marketplace
```

HandMirrorMcp는 다음과 같이 추가합니다.

```shell
claude mcp add HandMirrorMcp -- dnx HandMirrorMcp@0.1.1 --yes
```

### A-4. 바이브 코딩 활용 예시

두 MCP 서버가 연결된 상태에서, AI 에이전트에게 다음과 같이 요청할 수 있습니다.

**예시 1: 새 API를 사용한 코드 작성**

> "Microsoft.Data.Sqlite의 SqliteConnection 클래스가 제공하는 메서드를 확인하고, 트랜잭션을 사용한 배치 INSERT 코드를 작성해줘."

에이전트는 HandMirrorMcp의 `inspect_nuget_package_type`으로 실제 API를 확인한 뒤 코드를 생성합니다.

**예시 2: 빌드 오류 진단**

> "CS1061 오류가 발생하는데, ModelContextProtocol 패키지의 실제 메서드 시그니처를 확인해서 수정해줘."

에이전트는 HandMirrorMcp의 `explain_build_error`와 `inspect_nuget_package`를 조합하여 오류 원인을 파악합니다.

**예시 3: 공식 문서 기반 구현**

> ".NET 10 파일 기반 앱에서 launch profile을 설정하는 방법을 찾아서 app.run.json 파일을 만들어줘."

에이전트는 Microsoft Learn MCP 서버의 `microsoft_docs_search`로 공식 문서를 조회한 뒤, 정확한 형식으로 파일을 생성합니다.
