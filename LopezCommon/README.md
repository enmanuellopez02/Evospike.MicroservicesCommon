# MicroservicesCommon
This repository contains the code to build microservices with common codes in the shortest possible time.

[![Build Status](https://dev.azure.com/enmanuellopez02/MicroservicesCommon/_apis/build/status/MicroservicesCommon-ASP.NET%20Core-CI?branchName=main)](https://dev.azure.com/enmanuellopez02/MicroservicesCommon/_build/latest?definitionId=9&branchName=main)

### `appsettings.json` configuration

The file path and other settings can be read from JSON configuration if desired.

In `appsettings.json` add a `"MongoDbSettings", "ServiceSettings", "RabbitMQSettings"` properties:

```json
{
   "MongoDbSettings": {
    "Host": "localhost",
    "Port": "27017"
  },
  "ServiceSettings": {
    "ServiceName": "Catalog",
    "Authority": "https://localhost:5003"
  },
  "RabbitMQSettings": {
    "Host": "localhost"
  }
}
```

And then pass the configuration section to the next methods:

```csharp
services.AddMongo()
        .AddMongoRepository<Item>("YourTableName")
        .AddMassTransitWithRabbitMQ()
        .AddJwtBearerAuthetication();
```

Your model must inherit from IEntity class

```csharp
public class Item : IEntity
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public DateTimeOffset CreatedDate { get; set; }
}
```

Example of a controller using dependency injection services

```csharp
[Route("[controller]")]
[ApiController]
public class ItemsController : ControllerBase
{
    private const string AdminRole = "Admin";
    private readonly IRepository<Item> _itemsRepository;
    private readonly IPublishEndpoint _publishEndpoint;

    public ItemsController(IRepository<Item> itemsRepository, IPublishEndpoint publishEndpoint)
    {
        _itemsRepository = itemsRepository;
        _publishEndpoint = publishEndpoint;
    }

    [HttpGet]
    [Authorize(Policies.Read)]
    public async Task<IEnumerable<ItemDto>> GetAsync()
    {
        var items = (await _itemsRepository.GetAllAsync()).Select(item => item.AsDto());
        return items;
    }

    [HttpGet("{id}")]
    [Authorize(Policies.Read)]
    public async Task<ActionResult<ItemDto>> GetByIdAsync(Guid id)
    {
        var item = await _itemsRepository.GetAsync(id);

        if(item == null)
        {
            return NotFound();
        }

        return item.AsDto();
    }

    [HttpPost]
    [Authorize(Policies.Write)]
    public async Task<ActionResult<ItemDto>> PostAsync(CreateItemDto createItemDto)
    {
        var item =  new Item
        {
            Name = createItemDto.Name,
            Description = createItemDto.Description,
            Price = createItemDto.Price,
            CreatedDate = DateTimeOffset.UtcNow
        };

        await _itemsRepository.CreateAsync(item);
        await _publishEndpoint.Publish(new CatalogItemCreated(item.Id, item.Name, item.Description));

        return CreatedAtAction(nameof(GetByIdAsync), new { id = item.Id}, item);
    }

    [HttpPut("{id}")]
    [Authorize(Policies.Write)]
    public async Task<IActionResult> PutAsync(Guid id, UpdateItemDto updateItemDto)
    {
        var existingItem = await _itemsRepository.GetAsync(id);

        if(existingItem == null)
        {
            NotFound();
        }

        existingItem.Name = updateItemDto.Name;
        existingItem.Description = updateItemDto.Description;
        existingItem.Price = updateItemDto.Price;

        await _itemsRepository.UpdateAsync(existingItem);
        await _publishEndpoint.Publish(new CatalogItemUpdated(existingItem.Id, existingItem.Name, existingItem.Description));

        return NoContent();
    }

    [HttpDelete("{id}")]
    [Authorize(Policies.Write)]
    public async Task<IActionResult> DeleteAsync(Guid id)
    {
        var existingItem = await _itemsRepository.GetAsync(id);

        if (existingItem == null)
        {
            NotFound();
        }

        await _itemsRepository.RemoveAsync(existingItem.Id);
        await _publishEndpoint.Publish(new CatalogItemDeleted(existingItem.Id));

        return NoContent();
    }
}
```

Example of a consumer used by RabbitMQ to consume posts from other microservices

```csharp
public class CatalogItemCreatedConsumers : IConsumer<CatalogItemCreated>
{
    private readonly IRepository<CatalogItem> _catalogItemsRepository;

    public CatalogItemCreatedConsumers(IRepository<CatalogItem> catalogItemsRepository)
    {
        _catalogItemsRepository = catalogItemsRepository;
    }

    public async Task Consume(ConsumeContext<CatalogItemCreated> context)
    {
        var message = context.Message;
        var item = await _catalogItemsRepository.GetAsync(message.ItemId);

        if(item != null)
        {
            return;
        }

        item = new CatalogItem
        {
            Id = message.ItemId,
            Name = message.Name,
            Description = message.Description
        };

        await _catalogItemsRepository.CreateAsync(item);
    }
}
```