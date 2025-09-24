- var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
+ var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw, JsonUtf));

- return JsonNode.Parse(JsonSerializer.Serialize(pdRaw)) as JsonObject;
+ return JsonNode.Parse(JsonSerializer.Serialize(pdRaw, JsonUtf)) as JsonObject;
