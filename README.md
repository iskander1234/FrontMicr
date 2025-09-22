
BuildClassificationVariables(...)


camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest {
    TaskId = claimed.TaskId,
    Variables = stageVars   // здесь внутри уже есть classification = "Normal"
});


// вход: Dictionary<string, object> variables
var camundaVars = new Dictionary<string, object>();

foreach (var (key, val) in variables)
{
    if (val is null) continue;

    string type = val switch
    {
        bool   => "Boolean",
        int    => "Integer",
        long   => "Long",
        double => "Double",
        decimal=> "Double", // или "Number" если свой бэкенд
        DateTime => "Date",
        _      => "String"
    };

    camundaVars[key] = new { value = val, type };
}

// Итоговый POST должен выглядеть так:
var body = new
{
    variables = camundaVars
    // withVariablesInReturn = false  // опционально
};

// POST /task/submit?taskId=...
