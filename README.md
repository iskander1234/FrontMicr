var evt = new DelegationRemovedEvent
  {
      EntityId          = entity.Id,
      PrincipalUserCode = command.PrincipalCode,
      DeputyUserCode    = command.DeputyCode
  };
