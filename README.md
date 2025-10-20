// ИСПРАВЛЕННЫЙ метод CreateIgCreativeAsync

private async Task<string> CreateIgCreativeAsync(
    HttpClient http, string actId, CreateVideoAdRequest r, string videoId, string igUserId,
    bool wantsVideo, CancellationToken ct)
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    void Trace(string m) => _log.LogInformation("IGCreative STEP {Ms}ms :: {Msg}", sw.ElapsedMilliseconds, m);

    // === CRITICAL FIX #1: Гарантировать imageHash ПЕРЕД линком ===
    Trace("EnsureThumbHashAsync (aggressive) start");
    string? imageHash = null;
    
    // Если есть ThumbnailUrl юзера — начнём с него
    if (!string.IsNullOrWhiteSpace(r.ThumbnailUrl))
    {
        try
        {
            imageHash = await UploadAdImageFromUrlAsync(http, actId, r.ThumbnailUrl!, ct);
            if (!string.IsNullOrWhiteSpace(imageHash))
            {
                await WaitAdImageIndexedAsync(http, actId, imageHash!, TimeSpan.FromSeconds(30), ct);
                Trace($"EnsureThumbHashAsync done (from ThumbnailUrl) hash={imageHash.Substring(0, Math.Min(8, imageHash.Length))}");
            }
        }
        catch (Exception ex)
        {
            _log.LogWarning(ex, "Upload from ThumbnailUrl failed, will try video thumb");
        }
    }

    // Если ThumbnailUrl не помог — ждём превью видео (более агрессивно)
    if (string.IsNullOrWhiteSpace(imageHash))
    {
        // НОВОЕ: retry с экспоненциальной задержкой, максимум 45 сек
        var thumbTimeout = TimeSpan.FromSeconds(45);
        var thumbDeadline = DateTime.UtcNow.AddSeconds(45);
        int thumbAttempt = 0;

        while (DateTime.UtcNow < thumbDeadline && string.IsNullOrWhiteSpace(imageHash))
        {
            thumbAttempt++;
            var videoThumbUrl = await TryGetVideoThumbUrlAsync(http, videoId, ct);
            
            if (!string.IsNullOrWhiteSpace(videoThumbUrl))
            {
                try
                {
                    imageHash = await UploadAdImageFromUrlAsync(http, actId, videoThumbUrl!, ct);
                    if (!string.IsNullOrWhiteSpace(imageHash))
                    {
                        await WaitAdImageIndexedAsync(http, actId, imageHash!, TimeSpan.FromSeconds(20), ct);
                        Trace($"EnsureThumbHashAsync done (from video thumb, attempt {thumbAttempt}) hash={imageHash.Substring(0, Math.Min(8, imageHash.Length))}");
                        break;
                    }
                }
                catch (Exception ex)
                {
                    _log.LogWarning(ex, "Upload from video thumb (attempt {Attempt}) failed", thumbAttempt);
                }
            }

            // Экспоненциальная задержка: 2, 4, 8, 16 сек
            var delayMs = Math.Min(2000 * (int)Math.Pow(2, thumbAttempt - 1), 16000);
            if (DateTime.UtcNow.AddMilliseconds(delayMs) < thumbDeadline)
            {
                await Task.Delay(delayMs, ct);
            }
            else break;
        }
    }

    // === CRITICAL FIX #2: Если нет imageHash, НЕ идём в link_data ===
    if (string.IsNullOrWhiteSpace(imageHash))
    {
        var errMsg = wantsVideo
            ? "Не удалось получить превью видео для Instagram (45+ сек ожидания). " +
              "Укажите r.ThumbnailUrl или переподогрейте видео."
            : "Нет картинки для link_data. Укажите r.ThumbnailUrl или используйте видео.";
        
        _log.LogError("CRITICAL: imageHash is empty. wantsVideo={WantsVideo}. {Msg}", wantsVideo, errMsg);
        throw new InvalidOperationException(errMsg);
    }

    // === ДИАГНОСТИКА ===
    Trace("Preflight start");
    async Task DiagnoseIgWiringAsync()
    {
        var igResp = await http.GetAsync($"{MetaApi.GraphBase}/{igUserId}?fields=id,username", ct);
        var igBody = await igResp.Content.ReadAsStringAsync(ct);
        _log.LogInformation("Preflight IG resp {Code}: {Body}", (int)igResp.StatusCode, MetaApi.Redact(igBody));
        igResp.EnsureSuccessStatusCode();

        var pageResp = await http.GetAsync($"{MetaApi.GraphBase}/{r.PageId}?fields=instagram_business_account", ct);
        var pageBody = await pageResp.Content.ReadAsStringAsync(ct);
        _log.LogInformation("Preflight Page resp {Code}: {Body}", (int)pageResp.StatusCode, MetaApi.Redact(pageBody));
        pageResp.EnsureSuccessStatusCode();

        try
        {
            using var doc = JsonDocument.Parse(pageBody);
            if (doc.RootElement.TryGetProperty("instagram_business_account", out var iba) &&
                iba.TryGetProperty("id", out var idProp) &&
                idProp.GetString() == igUserId)
            {
                Trace("Preflight OK (page↔ig linked)");
                return;
            }
        }
        catch { }

        _log.LogWarning("Preflight FAIL: page {PageId} does not link to ig {IgId}", r.PageId, igUserId);
    }
    await DiagnoseIgWiringAsync();

    // === KREATIVE BUILDING ===
    const string PrimaryIgKey = "instagram_actor_id";
    const string FallbackIgKey = "instagram_user_id";

    Dictionary<string, object?> BuildVideoOss(string igKey)
    {
        return new Dictionary<string, object?>
        {
            ["page_id"] = r.PageId!,
            [igKey] = igUserId,
            ["video_data"] = new Dictionary<string, object?>
            {
                ["video_id"] = videoId,
                ["message"] = r.PrimaryText ?? string.Empty,
                ["call_to_action"] = new Dictionary<string, object?>
                {
                    ["type"] = "INSTAGRAM_MESSAGE",
                    ["value"] = new Dictionary<string, object?> { ["app_destination"] = "INSTAGRAM_DIRECT" }
                },
                ["image_hash"] = imageHash  // ВСЕГДА включаем превью
            }
        };
    }

    async Task<Dictionary<string, object?>> BuildLinkOssAsync(string igKey)
    {
        string link = "https://www.instagram.com/";
        try
        {
            var igResp = await http.GetAsync($"{MetaApi.GraphBase}/{igUserId}?fields=username", ct);
            if (igResp.IsSuccessStatusCode)
            {
                using var igDoc = JsonDocument.Parse(await igResp.Content.ReadAsStringAsync(ct));
                if (igDoc.RootElement.TryGetProperty("username", out var u) && u.ValueKind == JsonValueKind.String)
                    link = $"https://www.instagram.com/{u.GetString()}/";
            }
        }
        catch { }

        return new Dictionary<string, object?>
        {
            ["page_id"] = r.PageId!,
            [igKey] = igUserId,
            ["link_data"] = new Dictionary<string, object?>
            {
                ["link"] = link,
                ["image_hash"] = imageHash,  // ОБЯЗАТЕЛЕН
                ["message"] = r.PrimaryText ?? string.Empty,
                ["call_to_action"] = new Dictionary<string, object?>
                {
                    ["type"] = "INSTAGRAM_MESSAGE",
                    ["value"] = new Dictionary<string, object?> { ["app_destination"] = "INSTAGRAM_DIRECT" }
                }
            }
        };
    }

    async Task<(bool ok, string? id, string body)> PostCreativeAsync(Dictionary<string, string> form)
    {
        using var resp = await http.PostAsync($"{MetaApi.GraphBase}/{actId}/adcreatives", 
            new FormUrlEncodedContent(form), ct);
        var body = await resp.Content.ReadAsStringAsync(ct);
        
        if (resp.IsSuccessStatusCode)
        {
            using var doc = JsonDocument.Parse(body);
            return (true, doc.RootElement.GetProperty("id").GetString(), body);
        }

        // === CRITICAL FIX #3: Гард против #1772101 ===
        var (code, sub, ut, um, blame, _, _) = ParseFbErrorRich(body);
        if (code == 1772101 || sub == 1772101)
        {
            _log.LogError("CRITICAL ERROR #1772101: No media in creative. OSS: {OSS}", 
                form.TryGetValue("object_story_spec", out var oss) ? oss : "<missing>");
            // Не ретраим — это системная ошибка
            return (false, null, body);
        }

        return (false, null, body);
    }

    // === ПОПЫТКА 1: ВИДЕО ===
    if (wantsVideo)
    {
        Trace("Build video OSS (actor)");
        var ossA = BuildVideoOss(PrimaryIgKey);
        var formA = new Dictionary<string, string>
        {
            ["name"] = r.AdName ?? "Creative IG (video)",
            ["object_story_spec"] = JsonSerializer.Serialize(ossA)
        };

        var (okA, idA, bodyA) = await PostCreativeAsync(formA);
        if (okA) 
        { 
            Trace("Creative OK (actor, video)"); 
            return idA!; 
        }

        _log.LogWarning("Creative(video, actor) failed: {Body}", MetaApi.Redact(bodyA));
        
        // Retry с instagram_user_id только если ошибка именно о ключе
        if (bodyA.Contains("instagram_actor_id", StringComparison.OrdinalIgnoreCase))
        {
            Trace("Retry with instagram_user_id");
            var ossU = BuildVideoOss(FallbackIgKey);
            var formU = new Dictionary<string, string>
            {
                ["name"] = r.AdName ?? "Creative IG (video, user fallback)",
                ["object_story_spec"] = JsonSerializer.Serialize(ossU)
            };
            var (okU, idU, bodyU) = await PostCreativeAsync(formU);
            if (okU) 
            { 
                Trace("Creative OK (user, video)"); 
                return idU!; 
            }
            
            if (r.VideoOnly)
                throw new InvalidOperationException($"Creative(IG video, both keys) failed: {MetaApi.Redact(bodyU)}");
        }
        else if (r.VideoOnly)
        {
            throw new InvalidOperationException($"Creative(IG video, actor) failed: {MetaApi.Redact(bodyA)}");
        }
    }

    // === ПОПЫТКА 2: LINK_DATA (НА ЭТОМ ЭТАПЕ imageHash ГАРАНТИРОВАН) ===
    Trace($"Build link OSS (actor) with imageHash={imageHash.Substring(0, Math.Min(8, imageHash.Length))}");
    var ossL = await BuildLinkOssAsync(PrimaryIgKey);
    var formL = new Dictionary<string, string>
    {
        ["name"] = r.AdName ?? "Creative IG (link_data)",
        ["object_story_spec"] = JsonSerializer.Serialize(ossL)
    };
    var (okL, idL, bodyL) = await PostCreativeAsync(formL);
    if (okL) 
    { 
        Trace("Creative OK (actor, link)"); 
        return idL!; 
    }

    _log.LogWarning("Creative(link, actor) failed: {Body}", MetaApi.Redact(bodyL));
    
    if (bodyL.Contains("instagram_actor_id", StringComparison.OrdinalIgnoreCase))
    {
        Trace("Retry link with instagram_user_id");
        var ossL2 = await BuildLinkOssAsync(FallbackIgKey);
        var formL2 = new Dictionary<string, string>
        {
            ["name"] = r.AdName ?? "Creative IG (link_data, user fallback)",
            ["object_story_spec"] = JsonSerializer.Serialize(ossL2)
        };
        var (okL2, idL2, bodyL2) = await PostCreativeAsync(formL2);
        if (okL2) 
        { 
            Trace("Creative OK (user, link)"); 
            return idL2!; 
        }
        throw new InvalidOperationException($"Creative(IG, link_data user) failed: {MetaApi.Redact(bodyL2)}");
    }

    throw new InvalidOperationException($"Creative(IG, link_data actor) failed: {MetaApi.Redact(bodyL)}");
}
