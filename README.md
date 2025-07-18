using BpmBaseApi.Domain.Entities.Event.Reference;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Reference;

/// <summary>
/// Департамент / отдел
/// </summary>
public class DepartmentEntity 
{
    public string Id { get; set; } = null!;
    public string Name { get; set; }
    public string? ManagerId { get; set; }
    public int Actual { get; set; }

    public string? ParentId { get; set; }

    public virtual DepartmentEntity? Parent { get; set; }
    public virtual ICollection<DepartmentEntity> InverseParent { get; set; } = new List<DepartmentEntity>();

    public virtual ICollection<DepartmentCreatedEvent> CreatedEvents { get; set; } =
        new List<DepartmentCreatedEvent>();

    public void Apply(DepartmentCreatedEvent @event)
    {
        Id = @event.Id;
        ParentId = @event.ParentId;
        Name = @event.Name;
        ManagerId = @event.ManagerId;
        Actual = @event.Actual;
    }
}


public abstract class BaseIntEntity
{
    public int Id { get; set; }
}
