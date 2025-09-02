if (request.Stage is ProcessStage.Completed or ProcessStage.Canceled)
{
    return new BaseResponseDto<SetProcessStageResponse>
    {
        Data = new SetProcessStageResponse { Status = "Ok" }
    };
}
