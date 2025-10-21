// ============================================
// ПАТЧ: ТОЛЬКО КРИТИЧЕСКИЕ ИСПРАВЛЕНИЯ
// ============================================
// Скопируйте эти методы в ваш AdCreationService.cs
// Замените существующие методы с такими же именами

// ============================================
// 1. RunVideoAdFlowAsync - ИЗМЕНИТЬ ТАЙМАУТ HttpClient
// ============================================
public async Task<CreateVideoAdResult> RunVideoAdFlowAsync(
    CreateVideoAdRequest r,
    CancellationToken ct = default)
{
    var auth = await GetValidToken(r.CompanyId, ct);
    var http = _httpFactory.CreateClient();
    http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", auth.Value);
    http.DefaultRequestHeaders.ExpectContinue = false;
    
    // ============ КРИТИЧНО: БЫЛО 30 МИНУТ, СТАЛО 3 МИНУТЫ ============
    http.Timeout = TimeSpan.FromMinutes(3);
    
    // ... остальной код метода БЕЗ ИЗМЕНЕНИЙ ...
    // Просто измените строку с http.Timeout
}

// ============================================
// 2. WaitVideoFullyWarmedAsync - СОКРАТИТЬ ВРЕМЯ ОЖИДАНИЯ
// ============================================
private async Task<VideoReadyInfo> WaitVideoFullyWarmedAsync(
    HttpClient http, string videoId, TimeSpan timeout, CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(timeout);

    var url = $"{MetaApi.GraphBase}/{videoId}?fields=" +
              "status{video_status,processing_phase,publishing_phase,processing_progress}," +
              "thumbnails{data{uri}}," +
              "format{width,height}," +
              "is_instagram_eligible";

    bool Ready(JsonDocument d, out bool igEligible, out (int w, int h)? dims)
    {
        igEligible = false;
        dims = null;
        var root = d.RootElement;

        bool stReady = false;
        if (root.TryGetProperty("status", out var st) && st.ValueKind == JsonValueKind.Object)
        {
            var videoStatus = GetPropStringSafe(st, "video_status");
            var processing = GetPropStringSafe(st, "processing_phase");
            stReady = string.Equals(videoStatus, "ready", StringComparison.OrdinalIgnoreCase)
                      && string.Equals(processing, "complete", StringComparison.OrdinalIgnoreCase);
        }

        if (root.TryGetProperty("is_instagram_eligible", out var ig))
        {
            var s = JsonElementToStringSafe(ig);
            igEligible = string.Equals(s, "true", StringComparison.OrdinalIgnoreCase);
        }

        if (root.TryGetProperty("format", out var fmts) && fmts.ValueKind == JsonValueKind.Array)
        {
            foreach (var f in fmts.EnumerateArray())
            {
                if (f.TryGetProperty("width", out var wEl) && wEl.ValueKind == JsonValueKind.Number &&
                    f.TryGetProperty("height", out var hEl) && hEl.ValueKind == JsonValueKind.Number)
                {
                    var w = wEl.GetInt32();
                    var h = hEl.GetInt32();
                    if (w > 0 && h > 0)
                    {
                        dims = (w, h);
                        break;
                    }
                }
            }
        }

        return stReady && dims is not null;
    }

    bool lastIgEligible = false;
    (int w, int h)? lastDims = null;
    int probe = 0;
    
    try
    {
        while (!cts.IsCancellationRequested)
        {
            using var r = await http.GetAsync(url, cts.Token);
            if (r.IsSuccessStatusCode)
            {
                using var doc = JsonDocument.Parse(await r.Content.ReadAsStringAsync(cts.Token));
                if (Ready(doc, out lastIgEligible, out lastDims))
                {
                    _log.LogInformation("✓ Video ready after {Probe} probes", probe);
                    await Task.Delay(TimeSpan.FromSeconds(2), cts.Token);
                    return new VideoReadyInfo(lastIgEligible, lastDims);
                }

                if (++probe % 5 == 0)
                {
                    var root = doc.RootElement;
                    string vs = "", ph = "";
                    if (root.TryGetProperty("status", out var st) && st.ValueKind == JsonValueKind.Object)
                    {
                        vs = GetPropStringSafe(st, "video_status") ?? "?";
                        ph = GetPropStringSafe(st, "processing_phase") ?? "?";
                    }

                    _log.LogInformation("⏳ Video probe {Probe}: status={VS}, phase={PH}", probe, vs, ph);
                }
            }

            await Task.Delay(TimeSpan.FromSeconds(1), cts.Token);
        }
    }
    catch (OperationCanceledException)
    {
        _log.LogError("❌ Video warmup TIMEOUT after {Probe} probes, {Timeout}s elapsed", 
            probe, timeout.TotalSeconds);
        throw new TimeoutException($"Видео не подготовилось за {timeout.TotalSeconds} сек");
    }

    throw new TimeoutException("Видео слишком долго подготавливается.");
}

// ============================================
// 3. ВЫЗОВ WaitVideoFullyWarmedAsync - ИЗМЕНИТЬ ПАРАМЕТР
// ============================================
// В методе RunVideoAdFlowAsync найдите строку:
// var readyInfo = await WaitVideoFullyWarmedAsync(http, videoId, TimeSpan.FromMinutes(8), ct);
// 
// ЗАМЕНИТЕ НА:
var readyInfo = await WaitVideoFullyWarmedAsync(http, videoId, TimeSpan.FromMinutes(2), ct);

// ============================================
// 4. ДОБАВИТЬ ТАЙМАУТЫ К ДОЛГИМ HTTP ЗАПРОСАМ
// ============================================

// TryGetVideoThumbUrlAsync - ДОБАВИТЬ ТАЙМАУТ
private async Task<string?> TryGetVideoThumbUrlAsync(HttpClient http, string videoId, CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(10)); // ← ДОБАВИТЬ ЭТУ СТРОКУ
    
    using var resp = await http.GetAsync($"{MetaApi.GraphBase}/{videoId}?fields=thumbnails{{uri}}", cts.Token); // ← ЗАМЕНИТЬ ct НА cts.Token
    if (!resp.IsSuccessStatusCode) return null;

    using var doc = JsonDocument.Parse(await resp.Content.ReadAsStringAsync(cts.Token)); // ← ЗАМЕНИТЬ ct НА cts.Token
    if (!doc.RootElement.TryGetProperty("thumbnails", out var th) ||
        th.ValueKind != JsonValueKind.Object ||
        !th.TryGetProperty("data", out var arr) ||
        arr.ValueKind != JsonValueKind.Array)
        return null;

    using var e = arr.EnumerateArray();
    if (!e.MoveNext()) return null;
    var first = e.Current;

    return first.TryGetProperty("uri", out var uriEl) && uriEl.ValueKind == JsonValueKind.String
        ? uriEl.GetString()
        : null;
}

// UploadAdImageFromUrlAsync - ДОБАВИТЬ ТАЙМАУТ
private async Task<string?> UploadAdImageFromUrlAsync(HttpClient http, string actId, string imageUrl,
    CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(15)); // ← ДОБАВИТЬ ЭТУ СТРОКУ
    
    using var form = new FormUrlEncodedContent(new Dictionary<string, string> { ["url"] = imageUrl });
    using var resp = await http.PostAsync($"{MetaApi.GraphBase}/{actId}/adimages", form, cts.Token); // ← ЗАМЕНИТЬ ct НА cts.Token
    if (!resp.IsSuccessStatusCode) return null;
    using var doc = JsonDocument.Parse(await resp.Content.ReadAsStringAsync(cts.Token)); // ← ЗАМЕНИТЬ ct НА cts.Token
    if (!doc.RootElement.TryGetProperty("images", out var images) ||
        images.ValueKind != JsonValueKind.Object) return null;
    foreach (var prop in images.EnumerateObject())
        if (prop.Value.TryGetProperty("hash", out var h) && h.ValueKind == JsonValueKind.String)
            return h.GetString();
    return null;
}

// UploadAdImageFromBytesAsync - ДОБАВИТЬ ТАЙМАУТ
private async Task<string?> UploadAdImageFromBytesAsync(HttpClient http, string actId, byte[] bytes,
    string fileName, CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(15)); // ← ДОБАВИТЬ ЭТУ СТРОКУ
    
    using var mp = new MultipartFormDataContent();
    mp.Add(new ByteArrayContent(bytes), "filename", fileName);
    using var resp = await http.PostAsync($"{MetaApi.GraphBase}/{actId}/adimages", mp, cts.Token); // ← ЗАМЕНИТЬ ct НА cts.Token
    if (!resp.IsSuccessStatusCode) return null;
    using var doc = JsonDocument.Parse(await resp.Content.ReadAsStringAsync(cts.Token)); // ← ЗАМЕНИТЬ ct НА cts.Token
    if (!doc.RootElement.TryGetProperty("images", out var images) ||
        images.ValueKind != JsonValueKind.Object) return null;
    foreach (var prop in images.EnumerateObject())
        if (prop.Value.TryGetProperty("hash", out var h) && h.ValueKind == JsonValueKind.String)
            return h.GetString();
    return null;
}

// ============================================
// 5. ДОБАВИТЬ ЛОГИРОВАНИЕ ДЛЯ ДИАГНОСТИКИ
// ============================================
// В начало RunVideoAdFlowAsync добавьте:
_log.LogInformation("🚀 START RunVideoAdFlowAsync: Goal={Goal}, PageId={Page}, VideoId={Vid}", 
    r.Goal, r.PageId, r.VideoId);

// После загрузки видео добавьте:
_log.LogInformation("📹 Video uploaded/resolved: {VideoId}", videoId);

// После ожидания видео:
_log.LogInformation("✓ Video warmed: eligible={Eligible}, dims={Dims}", 
    readyInfo.IgEligible, readyInfo.Dims);

// Перед созданием кампании:
_log.LogInformation("📝 Creating campaign...");

// Перед созданием creative:
_log.LogInformation("🎨 Creating creative: mode={Mode}, wantsVideo={Video}", mode, wantsVideo);

// В конце:
_log.LogInformation("✅ DONE RunVideoAdFlowAsync: CampaignId={CId}, AdId={Aid}", 
    campaignId, adId)
