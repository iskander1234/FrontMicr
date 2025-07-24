 // Сортировка для корректной постраничной выдачи по атрибуту
                var sortControl = new SortRequestControl();
                sortControl.SortKeys.Add(new System.DirectoryServices.Protocols.SortKey("sAMAccountName"));
                request.Controls.Add(sortControl);
