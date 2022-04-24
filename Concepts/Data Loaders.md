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

tricks: 
* you could also pass concatinated ids to access single query like this
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
