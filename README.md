# QRS + Decorator vs CQRS + MediatR in .NET
In projects built with DDD (Domain-Driven Design) and Clean Architecture, we often face important architectural decisions. One of them is how we manage commands and queries.

Today, I want to share an approach I’ve experimented with and found highly effective in enterprise scenarios:
QRS with the Decorator Pattern – without using MediatR.
It’s a clean and powerful alternative to the traditional MediatR-based approach.

Instead of relying on MediatR to handle commands and queries, we build a pipeline where commands pass through a chain of decorators (such as LoggingDecorator, ValidationDecorator, etc.) before reaching their respective handler.

Decorator examples:  
LoggingDecorator<TCommand, TResult>  
ValidatorDecorator<TCommand, TResult>  
ExceptionHandlerDecorator<TCommand, TResult>  

Why this approach?  
✅ Full control over the command flow  
✅ No dependency on third-party libraries  
✅ Easier to test and debug  
✅ Better performance under high throughput  
✅ Advantages of using Decorator without MediatR:  
No external dependencies  
More flexible for enterprise systems with strict architectural policies  
Works great with event sourcing, audit logs, validations, and more  

⚠️ Disadvantages:  
More boilerplate code in the beginning  
You need to implement your own dispatcher (but only once)  

On GitHub, you can find a practical example of how I applied the CQRS with Decorator Pattern in a DDD and Clean Architecture structured project.
I also used a Static Factory Method for entity creation, which helps centralize business logic and maintain domain integrity.
This combination makes the code cleaner, more testable, and easier to maintain.

```csharp
// ===== Controller =====
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly ICommandDispatcher _commandDispatcher;

    public UsersController(ICommandDispatcher commandDispatcher)
    {
        _commandDispatcher = commandDispatcher;
    }

    [HttpPost]
    public async Task<IActionResult> Create(CreateUserRequest request)
    {
        var command = CreateUserCommand.Create(request.FirstName, request.LastName, request.Email);
        var result = await _commandDispatcher.SendAsync(command);
        return Ok(result);
    }
}

// ===== Application Layer =====
public record CreateUserCommand(string FirstName, string LastName, string Email) : ICommand<Guid>
{
    public static CreateUserCommand Create(string firstName, string lastName, string email)
    {
        // Këtu mund të bëhet validim bazik ose normalizim në të ardhmen
        return new CreateUserCommand(firstName, lastName, email);
    }
}

public class CreateUserCommandHandler : ICommandHandler<CreateUserCommand, Guid>
{
    private readonly IUserRepository _userRepository;

    public CreateUserCommandHandler(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }

    public async Task<Guid> HandleAsync(CreateUserCommand command)
    {
        var user = Domain.Entities.User.Create(command.FirstName, command.LastName, command.Email);
        await _userRepository.AddAsync(user);
        return user.Id;
    }
}

// ===== Domain Layer =====
namespace Domain.Entities
{
    public class User : BaseEntity
    {
        public string FirstName { get; private set; }
        public string LastName { get; private set; }
        public string Email { get; private set; }

        private User(Guid id, string firstName, string lastName, string email)
        {
            Id = id;
            FirstName = firstName;
            LastName = lastName;
            Email = email;
        }

        public static User Create(string firstName, string lastName, string email)
        {
            var id = Guid.NewGuid();
            // Mund të shtosh validime specifike të domenit këtu
            return new User(id, firstName, lastName, email);
        }
    }
}

// ===== Infrastructure Layer =====
public interface IUserRepository
{
    Task AddAsync(User user);
    // Opsione tjera: GetById, Remove, Update
}

// ===== Decorator Layer =====
public class LoggingDecorator<TCommand, TResult> : ICommandHandler<TCommand, TResult> where TCommand : ICommand<TResult>
{
    private readonly ICommandHandler<TCommand, TResult> _inner;
    private readonly ILogger<LoggingDecorator<TCommand, TResult>> _logger;

    public LoggingDecorator(ICommandHandler<TCommand, TResult> inner, ILogger<LoggingDecorator<TCommand, TResult>> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<TResult> HandleAsync(TCommand command)
    {
        _logger.LogInformation("Handling command {Command}", typeof(TCommand).Name);
        var result = await _inner.HandleAsync(command);
        _logger.LogInformation("Handled command {Command}", typeof(TCommand).Name);
        return result;
    }
}

public class ValidationDecorator<TCommand, TResult> : ICommandHandler<TCommand, TResult> where TCommand : ICommand<TResult>
{
    private readonly ICommandHandler<TCommand, TResult> _inner;
    private readonly IEnumerable<IValidator<TCommand>> _validators;

    public ValidationDecorator(ICommandHandler<TCommand, TResult> inner, IEnumerable<IValidator<TCommand>> validators)
    {
        _inner = inner;
        _validators = validators;
    }

    public async Task<TResult> HandleAsync(TCommand command)
    {
        foreach (var validator in _validators)
        {
            var validationResult = await validator.ValidateAsync(command);
            if (!validationResult.IsValid)
            {
                throw new ValidationException(validationResult.Errors);
            }
        }

        return await _inner.HandleAsync(command);
    }
}

// ===== DI Setup =====
services.AddScoped<IUserRepository, UserRepository>();
services.AddScoped<ICommandHandler<CreateUserCommand, Guid>, CreateUserCommandHandler>();
services.Decorate<ICommandHandler<CreateUserCommand, Guid>, ValidationDecorator<CreateUserCommand, Guid>>();
services.Decorate<ICommandHandler<CreateUserCommand, Guid>, LoggingDecorator<CreateUserCommand, Guid>>();




#dotnet #cleanarchitecture #cqrs #decoratorpattern #ddd #architecture #softwaredesign #aspnetcore #backend
