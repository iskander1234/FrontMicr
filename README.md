[HttpPost("send")]
[SwaggerResponse(StatusCodes.Status200OK, "Отправлено", typeof(BaseResponseDto<SendProcessResponse>))]
public async Task<IActionResult> SendItsmAsync([FromBody] SendProcessCommand command, CancellationToken ct)
{
    var result = await mediator.Send(command, ct);
    return Ok(result);
}


{
  "taskId": "PUT-YOUR-TASK-GUID-HERE",
  "action": "Submit",
  "payloadJson": {
    "itsmLevel": "Emergency"
  }
}


{
  "taskId": "PUT-YOUR-TASK-GUID-HERE",
  "action": "Submit"
}
