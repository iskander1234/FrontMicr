var delegations = await unitOfWork.DelegationRepository
    .GetByFilterListAsync(cancellationToken,
        d => d.DeputyUserCode.ToLower() == query.UserCode.ToLower());
