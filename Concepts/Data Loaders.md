## How to use DataLoaders ?

1. add dataloader factory 
```js
const dataLoaderFactory = <T1, T2>(dlFunc) =>
  new DataLoader<T1, T2>(dlFunc, { cache: true });

```

2. declare data loader variable in data source 
```js
  private loaders?: { [key: string]: DataLoader<string, CityDocument> };

```
3. initialize loader inside DataSource
```js
initialize(config) {
    this.context = config.context;
    this.loaders = {
      Cities: dataLoaderFactory(async (ids) => {
        const results = await this.context.models.City.find({
          _id: { $in: ids },
        }).lean();
        return convertToDataloaderResult(ids, results, "_id");
      })
    }
}
```

4. add function to convert results to dataloader result format
```js
function convertToDataloaderResult(requestedIds, returnedItems, key = "id") {
  const byId = _.keyBy(returnedItems, key);
  return requestedIds.map((id) => (byId[id] === undefined ? null : byId[id]));
}
```

5. finally define function to use dataloaders
```js
 find(id: string) {
    if (!id) throw new Error("Can not find city without id");
    return this.loaders.Cities.load(id);
  }

  findMany(ids) {
    if (!ids) throw new Error("Can not find Cities without ids");
    return this.loaders.Cities.loadMany(ids);
  }
```

Tricks: 
=======
* you could also pass concatinated ids to access single query like this
ex: storeId-cityId ===> rwekljhfkgndfgfdgd-mnolklvxmnzolklwke1s5d
```js
citiesByStateId: dataLoaderFactory(async (ids) => {
        const stateIds = [];
        const storeId = ids[0]?.split("-")[0];
        ids.forEach((id) => {
          stateIds.push(id.split("-")[1]);
        });
        const results = await getPaginatedResponseMongoose(
          this.context.models.City,
          {
            $or: [{ storeId }, { storeId: { $exists: false } }],
            stateId: { $in: stateIds },
          },
          { includeHasNext: true, includeHasPrev: true }
        ).mongoosePagination({ first: 10000 });

        const groupByStateId = _.groupBy(results.nodes, "stateId");
        const mappedResult = stateIds.map((stateId, index) => ({
          _id: ids[index],
          nodes: groupByStateId[stateId],
          totalCount: groupByStateId[stateId]?.length || 0,
        }));
        return convertToDataloaderResult(ids, mappedResult, "_id");
      })
```
FAQ
===

### When to use DataLoader and when to use Redis ?

DataLoader is primarily a means of batching requests to some data source. However, it does optionally utilize caching on a per request basis. This means, while executing the same GraphQL query, you only ever fetch a particular entity once. For example, we can call load(1) and load(2) concurrently and these will be batched into a single request to get two entities matching those ids. If another field calls load(1) later on while executing the same request, then that call will simply return the entity with ID 1 we fetched previously without making another request to our data source.

DataLoader's cache is specific to an individual request. Even if two requests are processed at the same time, they will not share a cache. DataLoader's cache does not have an expiration nor a mechanism for invalidating individual cache values -- and it has no need to since the cache will be deleted once the request completes.

Redis is a key-value store that's used for caching, queues, PubSub and more. We can use it to provide response caching, which would let us effectively bypass the resolver for one or more fields and use the cached value instead (until it expires or is invalidated). We can use it as a cache layer between GraphQL and the database, API or other data source -- for example, this is what RESTDataSource does. We can use it as part of a PubSub implementation when implementing subscriptions.

