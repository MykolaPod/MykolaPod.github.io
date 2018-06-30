---
layout: post
title:  "Loop through related entities within Entity Framework"
date:   2018-06-30 08:29:54 +0300
categories: [.NET, EntityFramework]
tags: [.NET, EntityFramework]
---

Hi! Sometimes we faced with a situation when most reasonable solution to reach a goal is make some operation on all entities that are in relationship (database) with given entity. For example cascade deletion several sets of entities.
But what if given entity is a "child" and you need to delete both child and parents. O even entities do not have foreign key relationship.
Here I'd like to share with you my solution how to loop through all related entities and make an update using Entity Framework 6.

So what do we have:
Server with data (source), and target server.
During transfer data, on target side entity cannot be inserted/updated due to some reason. Target returns http response with an Id of problem entity.
What need to be done - "resync" entity and all related to it entitites.

Here is an example of source method that sends data:
{% highlight C# %}
var rawJson = JsonConvert.SerializeObject(
    entitiesForSend, 
    Formatting.None, 
    new JsonSerializerSettings { 
        NullValueHandling = NullValueHandling.Ignore, 
        ReferenceLoopHandling = ReferenceLoopHandling.Ignore, 
        ContractResolver = new IgnoreentityBasePropertiesContract(), 
        TypeNameHandling = TypeNameHandling.Objects, 
        Binder = new TypeNameSerializationBinder() 
        }
    ); 
endpoint = $"/api/{typeof(T).Name}Sync?isLast={isLast}"; 
response = await client.PutAsync(endpoint, new StringContent(ProcessJson(rawJson), Encoding.UTF8, "application/json")); 
if (response.StatusCode != HttpStatusCode.OK) 
{ 
    var responseBody = string.Empty; 
    try 
    { 
        responseBody = await response.Content.ReadAsStringAsync(); 
        if (response.StatusCode == HttpStatusCode.InternalServerError) 
        { 
            var problemEntitiesIds = ParseResponseBodyToIds(responseBody); 
            TryAddProblemEntitiesForTransferQueue(problemEntitiesIds); 
        } 
    } 
    catch (Exception ex) 
    { 
        await _logService.Exception(
            ServerType.Source, 
            target.Name, 
            "Re-queue items for transfer failed", 
            type: typeof(T).Name, 
            exception: $"Response Code: {response.StatusCode}, 
            body: {responseBody}, 
            endpoint: {endpoint}, 
            exception: {ex.Message}"
            ); 
    }

    throw new DataTransferException(); 
}
{% endhighlight %}

Here is we have a list of objects (List<T> where T : EntityBase) that need to be send `entitiesForSend`, we'r sending it to target and handling response. There is nothing special yet. But need to know that there is several types that may be synced.

Assume that target side is parsing request and inserting\updating transfered entities at DB. And due to some reason insert\update failed. Target sending back to source response with Id of all entities processing of which has been failed. 

At Source side it looks like `response.StatusCode == HttpStatusCode.InternalServerError`. It's parsing a response and passing `problemEntitiesIds` to `TryAddProblemEntitiesForTransferQueue`.

Here we'r having a list of id of entities that has not been synced with target. In order to make it sync we need to sync all related entities from down to top (parent to child chain) - that will fix integrity of data on target side for current set of entities.

In our example in order to sync entity we need set `ModifiedDateTime` entity property to current DateTime - and entity will be in a queue to sync for next time.
Let's take a look closer how we can loop through all related (to current) entities chain and update `ModifiedDateTime` for all of them.

<i>I've put it in Context class</i>
{% highlight C# %}
// <summary> 
/// Get all entities that are in relationship with TEntity 
/// </summary> 
/// <typeparam name="TEntity">entity type fogerin keys which need to be find</typeparam> 
/// <returns></returns> 
public virtual IEnumerable<string> GetForeginKeysOf<TEntity>() where TEntity : class 
{ 
    IEnumerable<string> foreginKeys; 
    var objectContext = ((IObjectContextAdapter)this).ObjectContext; 
    var fullNameTypes = objectContext.MetadataWorkspace.GetItems<EntityType>(DataSpace.OSpace); 
    var currentTypeMetadata = fullNameTypes.FirstOrDefault(types => types.FullName == typeof(TEntity).FullName); 
    if (currentTypeMetadata != null) 
    { 
        foreginKeys = currentTypeMetadata.NavigationProperties.Select(n => n.Name); 
    } 
    else 
    { 
        foreginKeys = new List<string>(); 
    } 

    return foreginKeys; 
}
{% endhighlight %}
This allow look thtough EntityFramework metadata of particular type and get name of entity properties that are foregin keys to others.

Let's go back to our source. In case of sync failure we used `TryAddProblemEntitiesForTransferQueue` method:
{% highlight C# %}
private void TryAddProblemEntitiesForTransferQueue(List<Guid> exceptionEntities) 
{ 
    //repository is IRepository<T> (where T is a type of current entity that has been unsuccessfully transfered) 
    var dbEntities = _repository.All.Where(e => exceptionEntities.Contains(e.Id)).ToList(); 
    if (!dbEntities.Any()) 
    { 
        return; 
    } 
    var context = _repository.Context; 
    List<string> foreginKeys = context.GetForeginKeysOf<T>().ToList(); 
    // queue curent problem item itself _repository.BatchInsertOrUpdate(dbEntities);
    // update modified datetime _repository.Save(); 
    var contextProps = context.GetType().GetProperties(); 
    foreach (var propertyName in foreginKeys) 
    { 
        var propertyOfSubEntity = GetPropertyInfo(propertyName); 
        if (propertyOfSubEntity == null || !propertyOfSubEntity.GetGetMethod().IsVirtual) 
        { 
            continue; 
        } 
        PropertyInfo dbSetProp = null; 
        foreach (var propertyInfo in contextProps) 
        { 
            if (propertyInfo.PropertyType.IsGenericType && propertyInfo.PropertyType.GetGenericTypeDefinition() == typeof(IDbSet<>)) 
            { 
                var dbSettype = propertyInfo.PropertyType.GetGenericArguments().First(); 
                if (dbSettype == propertyOfSubEntity.PropertyType) 
                { 
                    dbSetProp = propertyInfo; 
                    break; 
                } 
            } 
        } 
        if (dbSetProp == null) 
        { 
            continue; 
        } 
        foreach (var problemEntity in dbEntities) 
        { 
            var subEntityIdValue = propertyOfSubEntity.GetValue(problemEntity) as EntityBase; 
            if (subEntityIdValue == null) 
            { 
                continue; 
            } 
            var resultId = subEntityIdValue.Id; 
            SwitchOverTransferTypes(propertyOfSubEntity.PropertyType.Name, context, resultId); 
        } 
        context.SaveChanges(); 
    } 
}
{% endhighlight %}

{% highlight C# %}
private PropertyInfo GetPropertyInfo(string propertyName) 
{ 
    //truncate Id at the end of name to get name of entity property if need 
    // assumed that attribute foreign key is not used. If needed - add handling such case here 
    var propertyOfSubEntity = typeof(T).GetProperty(propertyName); 
    return propertyOfSubEntity; 
}
{% endhighlight %}

In my case I have entity types that are inherited from base types: in order to be able to update every of them, we need specify all entity types that are may be synced - that will allow us to get dataset of each base type and update entity.

{% highlight C# %}
/// <summary> 
/// Find and call queue for appropriate dbSet depending of entity type that should be updated 
/// </summary> 
/// <param name="typeName">Type of related entity</param> 
/// <param name="context">Db context</param> 
/// <param name="resultId"> id of entity that sould be updated</param> 
private void SwitchOverTransferTypes(string typeName, IDbContext context, Guid resultId) 
{ 
    switch (typeName) 
    { 
        case nameof(ItemTypeBase): 
        case nameof(ItemType): 
        case nameof(CustomItemType): 
            QueueForTransfer<ItemTypeBase>(context, resultId); 
            break;
        case nameof(SerializedEntityBase): 
        case nameof(Item): 
        case nameof(ItemGroup): 
            QueueForTransfer<SerializedEntityBase>(context, resultId); 
            break; 
        case nameof(ProcessBase): 
        case nameof(ProcessSnapshot): 
        case nameof(ProcessTemplate): 
            QueueForTransfer<ProcessBase>(context, resultId); 
            break; 
        case nameof(ProcessStepBase): 
        case nameof(PassFailProcessStep): 
        case nameof(AssignItemToStandardItemTypeProcessStep): 
        case nameof(SetCustomPropertyProcessStep): 
        case nameof(SetGPSLocationStep): 
        case nameof(AssignItemToCustomItemTypeProcessStep): 
        case nameof(WithPictureStep):
        case nameof(SetDateCustomPropertyProcessStep): 
        case nameof(SetNumericCustomPropertyProcessStep): 
        case nameof(SetTextCustomPropertyProcessStep): 
        case nameof(TakePictureStep): 
        case nameof(CaptureSignatureStep): 
            QueueForTransfer<ProcessStepBase>(context, resultId); 
            break; 
        case nameof(CompletedProcessStepBase): 
        case nameof(AssignItemToCustomItemTypeCompletedProcessStep): 
        case nameof(WithPictureCompletedStep): 
        case nameof(PassFailCompletedProcessStep): 
        case nameof(SetNumericCustomPropertyCompletedProcessStep): 
        case nameof(SetDateCustomPropertyCompletedProcessStep): 
        case nameof(SetCustomPropertyCompletedProcessStep): 
        case nameof(SetGPSLocationCompletedStep): 
        case nameof(AssignItemToStandardItemTypeCompletedProcessStep): 
        case nameof(TakePictureCompletedStep): 
        case nameof(CaptureSignatureCompletedStep): 
            QueueForTransfer<CompletedProcessStepBase>(context, resultId); 
            break; 
        case nameof(CustomPropertyBase): 
        case nameof(ItemCustomProperty): 
        case nameof(ItemTypeCustomProperty): 
        case nameof(JobCustomProperty): 
            QueueForTransfer<CustomPropertyBase>(context, resultId); 
            break;
        case nameof(ProcessPhase): 
            QueueForTransfer<ProcessPhase>(context, resultId); 
            break; 
        case nameof(Picture): 
            QueueForTransfer<Picture>(context, resultId); 
            break; 
        case nameof(ItemGroupType): 
            QueueForTransfer<ItemGroupType>(context, resultId); 
            break; 
        case nameof(Job): 
            QueueForTransfer<Job>(context, resultId); 
            break; 
        case nameof(ItemProcessBase): 
        case nameof(ItemProcess): 
        case nameof(ItemGroupProcess): 
            QueueForTransfer<ItemProcessBase>(context, resultId); 
            break; 
        default: 
            return; 
    } 
}

{% endhighlight %}

Switch above allow us to use generic update method
{% highlight C# %}
/// <summary> 
/// Set ModifiedOnServerUtc to current datetime utc. Nex time this items will be transfered 
/// </summary> 
/// <typeparam name="TEntity">type of dbset</typeparam> 
/// <param name="context">dbcontext with dbsets</param> 
/// <param name="id">entity id</param> 
private void QueueForTransfer<TEntity>(IDbContext context, Guid id) where TEntity : EntityBase, ITransferedEntity 
{ 
    var dbSet = context.Set<TEntity>(); 
    // assume that some type of entities (for example steps) related to same entity (item) 
    // using Find instead of FirstOrDefault in order to get Item from cache 
    var dbEntity = dbSet.Find(id); 
    if (dbEntity != null) 
    { 
        dbEntity.ModifiedOnServerUtc = DateTime.UtcNow;
        // set db entry object as Modified. Doing this due to perfomance reasons with turned off auto change tracking
        context.SetModified(dbEntity);
    }
}
{% endhighlight %}

That's it. Entity that we've received back from traget (problem entity) and all it's relatives will be updated in a way to re-sync all it's data chaing again and fix corrupted data integrity.

<i>
You may say that here is a tonn of ugly reflection code, but I'll say - go try do it better without touching database layer. Current solution sutisfied business requirements. 
And that's most important thing. But you may suggest your solution - my contacts near below.
Code is not pretend to super performance - but it's not needed because it's handling exception situation caused by miss-match data during sync between two servers.
In such way source and target are communicating in order to self-fix data transfer.
</i>

Get lucky !