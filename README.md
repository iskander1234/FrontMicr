 // 🔧 НОВОЕ: Execution — считаем позитивом и по Action, если Condition не задан
        case ProcessStage.Execution:
        {
            bool isPositive = condition switch
            {
                ProcessCondition.accept => true,
                ProcessCondition.remake or ProcessCondition.reject => false,
                _ => action == ProcessAction.Submit
            };
            return new Dictionary<string, object> { [varName] = isPositive }; // varName == "executed"
        }

        // 🔧 НОВОЕ: ExecutionCheck — аналогично
        case ProcessStage.ExecutionCheck:
        {
            bool isPositive = condition switch
            {
                ProcessCondition.accept => true,
                ProcessCondition.remake or ProcessCondition.reject => false,
                _ => action == ProcessAction.Submit
            };
            return new Dictionary<string, object> { [varName] = isPositive }; // varName == "checked"
        }
