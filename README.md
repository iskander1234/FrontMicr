    // 1) наши бизнес-ошибки — просто пробрасываем, чтобы не потерять стек/сообщение
    catch (HandlerException)
    {
        throw;
    }
    // 2) отмену нельзя заворачивать
    catch (OperationCanceledException)
    {
        throw;
    }
    // 3) всё остальное — заворачиваем в HandlerException с inner (см. конструктор ниже)
    catch (Exception ex)
    {
        // логирование ex тут, если нужно
        throw new HandlerException("Внутренняя ошибка при установке этапа процесса", ex, ErrorCodesEnum.Business);
    }
