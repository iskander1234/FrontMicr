// ... внутри Handle(...) перед CamundaSubmitTask

// Больше НЕ требуем classification:
var variables = new Dictionary<string, object>();

// Если вдруг пришла классификация — можно добавить, но это НЕ обязательно
var classification = ReadClassification(command.PayloadJson);
if (!string.IsNullOrWhiteSpace(classification))
{
    variables["classification"] = classification;
}

// Логика для Level1/Level2/Level3
if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
{
    switch (stage)
    {
        case ProcessStage.Level1:
        {
            // executed => true, toSecondLine => false
            var execution = ReadExecution(command.PayloadJson); // "executed" | "toSecondLine" | null
            bool line1 = string.Equals(execution, "executed", StringComparison.OrdinalIgnoreCase)
                ? true
                : string.Equals(execution, "toSecondLine", StringComparison.OrdinalIgnoreCase)
                    ? false
                    : true; // дефолт если не пришло - true
            variables["Line 1"] = line1;
            break;
        }
        case ProcessStage.Level2:
        {
            bool line2 = ReadLineBool(command.PayloadJson, 2) ?? true;
            variables["Line 2"] = line2;
            break;
        }
        case ProcessStage.Level3:
        {
            bool line3 = ReadLineBool(command.PayloadJson, 3) ?? true;
            variables["Line 3"] = line3;
            break;
        }
    }
}

var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
{
    TaskId    = claimed.TaskId,
    Variables = variables
});
