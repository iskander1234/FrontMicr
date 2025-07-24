// Сортировка для корректной постраничной выдачи по атрибуту
                var sortKey = new SortKey("sAMAccountName", false);
                var sortControl = new SortRequestControl(new[] { sortKey });
                request.Controls.Add(sortControl);
