### SolutionFetcher – Intelligent Problem Solver API



## Core Components


```
public class SearchHistory
{
    public int Id { get; set; }
    public string Query { get; set; }
    public string Category { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string ResultJson { get; set; }
}
```
To allow EFCore use SQLite effectively during Design time,  SQLitePCL.Batteries.Init() must be called. Namespace: (SQLitePCLRaw.bundle_green),
Download => 'SQLite/ Compact Toolbox Extension'. Data Source: SQLite Database File
---


---

### QueryClassifier

```
public class QueryClassifier
{
    public string Classify(string query)
    {
        query = query.ToLower();

        // Programming languages
        string[] languages = {
            "c#", ".net", "javascript", "typescript", "python", "java", "kotlin",
            "swift", "php", "ruby", "go", "rust", "dart", "flutter"
        };
        if (languages.Any(q => query.Contains(q))) return "software";

        // Frameworks
        string[] frameworks = {
            "asp.net", "blazor", "react", "angular", "vue",
            "django", "flask", "spring", "node", "express", "laravel"
        };
        if (frameworks.Any(q => query.Contains(q))) return "software";

        // Bugs, errors, packages, modules
        if (query.Contains("error") || query.Contains("exception")
            || query.Contains("bug") || query.Contains("install")
            || query.Contains("package") || query.Contains("dependency"))
            return "software";

        // Web design / UI / CMS / Dashboard
        if (query.Contains("figma") || query.Contains("ui") || query.Contains("ux")
            || query.Contains("dashboard") || query.Contains("wordpress")
            || query.Contains("cms") || query.Contains("template"))
            return "webdesign";

        // Networking
        if (query.Contains("network") || query.Contains("router")
            || query.Contains("ip") || query.Contains("wifi") || query.Contains("lan"))
            return "networking";

        // Electrical
        if (query.Contains("wire") || query.Contains("voltage")
            || query.Contains("circuit") || query.Contains("electri") || query.Contains("motor"))
            return "electrical";

        // Plumbing
        if (query.Contains("pipe") || query.Contains("toilet")
            || query.Contains("leak") || query.Contains("plumb") || query.Contains("heater"))
            return "plumbing";

        // Construction / Building
        if (query.Contains("cement") || query.Contains("concrete")
            || query.Contains("building") || query.Contains("block") || query.Contains("deck"))
            return "construction";

        return "general";
    }
}
```

---

### ExternalApiService

```

public class ExternalApiService
{
    private readonly HttpClient _http;

    public ExternalApiService(HttpClient http) => _http = http;

    public async Task<List<object>> FetchResources(string query, string category)
    {
        var results = new List<object>();

        if (category == "software" || category == "webdesign")
        {
            results.Add(await StackOverflow(query));
            results.Add(await GithubRepos(query));
            results.Add(await GithubGists(query));
            results.Add(await DevTo(query));
            results.Add(await NugetPackages(query));
            results.Add(await NpmPackages(query));
            results.Add(await YouTube(query));
            results.Add(await LinkedInJobs(query));
        }
        else
        {
            results.Add(await DuckDuckGo(query));
            results.Add(await Wikipedia(query));
        }

        return results;
    }

    private async Task<object> StackOverflow(string q) =>
        await _http.GetFromJsonAsync<object>($"https://api.stackexchange.com/2.3/search?order=desc&sort=activity&intitle={q}&site=stackoverflow");

    private async Task<object> GithubRepos(string q)
    {
        _http.DefaultRequestHeaders.UserAgent.ParseAdd("solution-fetcher");
        _http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", "YOUR_GITHUB_TOKEN");
        return await _http.GetFromJsonAsync<object>($"https://api.github.com/search/repositories?q={q}");
    }

    private async Task<object> GithubGists(string q)
    {
        _http.DefaultRequestHeaders.UserAgent.ParseAdd("solution-fetcher");
        return await _http.GetFromJsonAsync<object>($"https://api.github.com/search/code?q={q}");
    }

    private async Task<object> DevTo(string q) =>
        await _http.GetFromJsonAsync<object>($"https://dev.to/api/articles?per_page=5&tag={q}");

    private async Task<object> NugetPackages(string q) =>
        await _http.GetFromJsonAsync<object>($"https://api.nuget.org/v3/query?q={q}");

    private async Task<object> NpmPackages(string q) =>
        await _http.GetFromJsonAsync<object>($"https://registry.npmjs.org/-/v1/search?text={q}");

    private async Task<object> YouTube(string q)
    {
        string apiKey = "MOCK_YOUTUBE_KEY";
        return await _http.GetFromJsonAsync<object>($"https://youtube.googleapis.com/youtube/v3/search?part=snippet&q={q}&key={apiKey}");
    }

    private async Task<object> LinkedInJobs(string q)
    {
        string mockKey = "MOCK_LINKEDIN_KEY";
        return new { message = "LinkedIn Jobs API requires OAuth. Provide token.", query = q };
    }

    private async Task<object> DuckDuckGo(string q) =>
        await _http.GetFromJsonAsync<object>($"https://api.duckduckgo.com/?q={q}&format=json");

    private async Task<object> Wikipedia(string q) =>
        await _http.GetFromJsonAsync<object>($"https://en.wikipedia.org/api/rest_v1/page/summary/{q}");
}
```

---

### HistoryService

```
public class HistoryService
{
    private readonly AppDbContext _db;
    public HistoryService(AppDbContext db) => _db = db;

    public async Task Save(string query, string category, string resultJson)
    {
        _db.SearchHistories.Add(new SearchHistory
        {
            Query = query,
            Category = category,
            ResultJson = resultJson
        });
        await _db.SaveChangesAsync();
    }

    public async Task<List<SearchHistory>> Recent() =>
        await _db.SearchHistories.OrderByDescending(x => x.Timestamp).Take(20).ToListAsync();

    public async Task<List<SearchHistory>> Related(string category) =>
        await _db.SearchHistories
            .Where(x => x.Category == category)
            .OrderByDescending(x => x.Timestamp)
            .Take(20)
            .ToListAsync();
}


---

```
### SearchController

```

public class SearchController : ControllerBase
{
    private readonly QueryClassifier _classifier;
    private readonly ExternalApiService _external;
    private readonly HistoryService _history;

    public SearchController(QueryClassifier classifier, ExternalApiService external, HistoryService history)
    {
        _classifier = classifier;
        _external = external;
        _history = history;
    }

    [HttpGet]
    public async Task<IActionResult> Search(string q)
    {
        var category = _classifier.Classify(q);
        var results = await _external.FetchResources(q, category);

        string json = JsonSerializer.Serialize(results);
        await _history.Save(q, category, json);

        return Ok(new
        {
            query = q,
            category,
            results,
            recent = await _history.Recent(),
            related = await _history.Related(category)
        });
    }
}


```
---

## 🔧 Rate Limiting

To protect your API and external services:

###  Install NuGet

```
dotnet add package AspNetCoreRateLimit
```

### Configure in Program.cs

```
builder.Services.AddMemoryCache();
builder.Services.Configure<IpRateLimitOptions>(options =>
{
    options.GeneralRules = new List<RateLimitRule>
    {
        new RateLimitRule
        {
            Endpoint = "",
            Limit = 100, // max requests
            Period = "1m" // per 1 minute
        }
    };
});
builder.Services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();
builder.Services.AddInMemoryRateLimiting();
```

### Add Middleware

```
app.UseIpRateLimiting();





### 🖼 Visual Flow Diagram

```
User Query
    │
    ▼
QueryClassifier
    │
    ▼
ExternalApiService ─────────► APIs (StackOverflow, GitHub, Dev.to, YouTube, LinkedIn, NuGet, NPM, DuckDuckGo, Wikipedia)
    │
    ▼
HistoryService (Save query, results)
    │
    ▼
SearchController returns:
{

    query,
    category,
    results,
    recent,
    related
}
