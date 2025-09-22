
BuildClassificationVariables(...)


camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest {
    TaskId = claimed.TaskId,
    Variables = stageVars   // здесь внутри уже есть classification = "Normal"
});


