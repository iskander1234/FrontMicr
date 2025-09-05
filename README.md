/// <summary>Сохранить как черновик (как Start, но без Camunda; статус = Draft)</summary>
    [HttpPost("save")]
    [SwaggerResponse(StatusCodes.Status200OK, "Черновик сохранён", typeof(BaseResponseDto<StartProcessResponse>))]
    [SwaggerResponse(StatusCodes.Status400BadRequest)]
    [SwaggerResponse(StatusCodes.Status404NotFound)]
    [SwaggerResponse(StatusCodes.Status500InternalServerError)]
    public async Task<IActionResult> SaveProcessAsync([FromBody] SaveProcessCommand command, CancellationToken cancellationToken)
    {
        var result = await mediator.Send(command, cancellationToken);
        return Ok(result);
    }

    /// <summary>Список черновиков по InitiatorCode (опционально: фильтр по ProcessCode, пагинация)</summary>
    [HttpGet("drafts")]
    [SwaggerResponse(StatusCodes.Status200OK, "Список черновиков", typeof(BaseResponseDto<List<GetDraftListItemResponse>>))]
    public async Task<IActionResult> GetDraftsAsync(
        [FromQuery] string initiatorCode,
        [FromQuery] string? processCode,
        [FromQuery] int? skip,
        [FromQuery] int? take,
        CancellationToken cancellationToken)
    {
        var result = await mediator.Send(
            new GetDraftsByInitiatorQuery(initiatorCode, processCode, skip, take),
            cancellationToken);
        return Ok(result);
    }
