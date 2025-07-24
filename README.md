+        // теперь просто пролистываем всё дерево постранично
+        var employees = new List<LDAPEmployee>();
+        const int pageSize = 1000;
+        byte[] cookie = Array.Empty<byte>();
+
+        do
+        {
+            var request = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);
+
+            // (опционально) сортируем по sAMAccountName, чтобы страница не «прыгал»
+            var sortControl = new SortRequestControl(new[] {
+                new SortKey("sAMAccountName", null, false)
+            });
+            request.Controls.Add(sortControl);
+
+            var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };
+            request.Controls.Add(pageControl);
+
+            var response = (SearchResponse)connection.SendRequest(request);
+            _logger.LogInformation("Получена страница с {0} записями", response.Entries.Count);
+
+            foreach (SearchResultEntry entry in response.Entries)
+            {
+                employees.Add(new LDAPEmployee(entry));
+            }
+
+            cookie = response.Controls
+                        .OfType<PageResultResponseControl>()
+                        .FirstOrDefault()?.Cookie
+                     ?? Array.Empty<byte>();
+        }
+        while (cookie.Length > 0);
+
+        _logger.LogInformation("Всего записано из LDAP: {0}", employees.Count);
+        return employees;
