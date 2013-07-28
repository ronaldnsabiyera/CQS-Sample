# CQS**CQS (command–query separation)** - is a principle that defines data model of a system in terms of queries and commands:* **Commands** affecting state of object and provide a simple way to execute a single operation in terms of domain model.* **Queries** allows to query information from a data source without affecting it's state.More detailed explanation of CQS principle at [Wikipedia](http://en.wikipedia.org/wiki/Command%E2%80%93query_separation).This sample is an implementation of CQS for .NET applications.# Projects* **DataModel** - domain objects and abstractions for queries and commands. Have no references to any data-access specific libraries.* **EFDataAccess** - implementation of data-access layer on top of Entity Framework 5.0 and SQL Compact.* **TestApp** - test application to run queries and commands.* **UnitTests** - unit-test for queries and commands.# Usage## Setup1) **Register in IoC-container IQueryFactory -> new QueryFactory() binding.** You should pass lambda to resolve Queries as a parameter QueryFactory object. It may be lambda to resolve queries from IoC.2) **Register in IoC-container ICommandsFactory -> new CommandFactory() binding.** You should pass lambda to resolve Commands as a parameter CommandFactory object. It may be lambda to resolve commands from IoC.3) **Register queries and commands** in IoC.## Creating and executing queries### Creating query1) Create query interface, mark it with **IQuery** and place it to "" folder "**DataModel\Queries**":>public interface IActiveUsersQuery : IQuery>>{>>>	IEnumerable<User> Execute();>>}2) Implement the query. Query implementation depends on data-access method. In this sample Entity Framework 5.0 sample provided.All EF-specific things located at **EFDataAccess** project.To implement EF-specific queries "**EFQueryBase**" class created (__EFDataAccess\Core\EFQueryBase.cs__). Let's implement "**IActiveUsersQuery**" query:>public class ActiveUsersQuery : EFQueryBase<ApplicationDbContext>, IActiveUsersQuery>>{>>>	public ActiveUsersQuery(ApplicationDbContext context)>>>>		: base(context)>>>	{>>>	}>>>>>	public IEnumerable<User> Execute()>>>	{>>>>		return DbContext>>>>			.Users>>>>			.Where(x => x.IsActive)>>>>			.ToArray();>>>	}>>}3) Register the query in IoC:>Bind<IActiveUsersQuery\>().To<ActiveUsersQuery\>().InTransientScope();Now you can resolve interface of query from IoC and execute it.### Executing query1) Resolve **IQueryFactory**:> IQueryFactory queryFactory = Container.Current.Resolve<IQueryFactory>();2) Resolve query:>IEnumerable<User> activeUsers = queryFactory>>>	.ResolveQuery<IActiveUsersQuery>();3) Execute the query:>IEnumerable<User> activeUsers = queryFactory>>>	.ResolveQuery<IActiveUsersQuery>()>>>	.Execute();### Testing queryYou should test query implementation (not query interface:)).1) Create source collection(s) and mock the data source (EF's DbContext in our case):>IEnumerable<User> users = new[]>>{>>>	new User {Id = 1, FirstName = "User1", LastName = "User1", Email = "user1@email.com", IsActive = false},>>>	new User {Id = 2, FirstName = "User2", LastName = "User2", Email = "user2@email.com", IsActive = false},>>>	new User {Id = 3, FirstName = "User3", LastName = "User3", Email = "user3@email.com", IsActive = false}>>};>>>ApplicationDbContext dbContext = new Mock<ApplicationDbContext>("Test")>>>	.SetupDbContextData(x => x.Users, users)>>>	.Build();2) Instantiate the query with mocked data source and execute the query:>var query = new ActiveUsersQuery(dbContext);>>IEnumerable<User> result = query.Execute();3) Check the results:>CollectionAssert.AreEqual(users.Where(x => x.IsActive).ToArray(), result.ToArray());## Creating and executing commands### Creating command1) Create command interface, mark it with **ICommand** and place to "**DataModel\Commands**":>public class CreateUserCommand : ICommand>>{>>>	public string FirstName { get; protected set; }>>>	public string LastName { get; protected set; }>>>	public string Email { get; protected set; }>>>>	__// Output parameter's setters marked as public__>>>	public int Id { get; set; }>>>>>	public CreateUserCommand(string firstName, string lastName, string email)>>>	{>>>>		FirstName = firstName;>>>>		LastName = lastName;>>>>		Email = email;>>>	}>>}2) Implement the command. Command implementation as well as queries depends on data-access method. In this sample Entity Framework 5.0 sample provided.All EF-specific things located at **EFDataAccess** project.To implement EF-specific command "**EFCommandHandlerBase**" class created (__EFDataAccess\Core\EFQueryBase.cs__). Let's implement handler for "**CreateUserCommand**" command:>public class CreateUserCommandHandler : EFCommandHandlerBase<CreateUserCommand, ApplicationDbContext>, ICommandHandler<CreateUserCommand>>>{>>>	public CreateUserCommandHandler(ApplicationDbContext dbContext)>>>>		: base(dbContext)>>>	{>>>	}>>>>>	public override void Execute(CreateUserCommand command)>>>	{>>>>		int id = DbContext.Users.Any() == false ? 1 : DbContext.Users.Max(x => x.Id) + 1;>>>>		User user = new User>>>>		{>>>>>			Id = id,>>>>>			FirstName = command.FirstName,>>>>>			LastName = command.LastName,>>>>>			Email = command.Email,>>>>>			IsActive = false>>>>		};>>>>>>		DbContext.Users.Add(user);>>>>		DbContext.SaveChanges();>>>>>		command.Id = user.Id;>>>	}>>}3) Regiser the command in IoC:> Bind<ICommandHandler<CreateUserCommand\>\>().To<CreateUserCommandHandler\>().InTransientScope();### Executing command1) Resolve **ICommandsFactory**:>ICommandsFactory commandsFactory = Container.Current.Resolve<ICommandsFactory\>();2) Create Command object:> CreateUserCommand createUserCommand = new CreateUserCommand("...","...","...");3) Execute the command:>commandsFactory.ExecuteQuery(createUserCommand);### Testing commandYou should test command handler implementation, instead of testing command definition:1) Create source collection(s) and mock the data source (EF's DbContext in our case):>IEnumerable<User\> users = new User[] { };>>ApplicationDbContext dbContext = new Mock<ApplicationDbContext\>("Test")>>	.SetupDbContextData(x => x.Users, users)>>	.Build();2) Instantiate the command handler with mocked data source and execute the command:>CreateUserCommandHandler commandHandler = new CreateUserCommandHandler(dbContext);>>commandHandler.Execute(new DataModel.Commands.CreateUserCommand("User1", "User1", "user1@email.com"));3) Check the results:>User[] result = dbContext.Users.ToArray();>>Assert.AreEqual(1, result.Count());>>Assert.AreEqual(1, result[0].Id);>>Assert.AreEqual("User1", result[0].FirstName);>>Assert.AreEqual("User1", result[0].LastName);>>Assert.AreEqual("user1@email.com", result[0].Email);>>Assert.AreEqual(false, result[0].IsActive);