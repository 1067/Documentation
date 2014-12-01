API v2 is readonly api built over IQueryable provider over REST api.

**API V2 is for internal usage only! Please do not share it with outers! It could be changed without any notice!**

# Url format

https://targetprocess/api/v2/**{entity}**?where=**{where}**&select=**{select}**&take=**{take}**&skip=**{skip}**&orderBy=**{orderby}**&callback=**{callback}**

* **{entity}** - entity type name in single case (Userstory,Bug) - see [here](https://plan.tpondemand.com/api/v1/index/meta) for the list of available entity types
* **{where}** - filter expression over collection of entities
* **{select}** - result projection.
* **{skip}**, **{take}** - paging parameters, how many entities skip and take. If the result contains more entities, the response will have fields "next" or/and "prev" links to another pages.
* **{orderBy}** - orderings
* **{callback}** - [JSONP](https://en.wikipedia.org/wiki/JSONP) callback

# Selectors

***

*selector* = *expression* | *objectExpression*

*objectExpression* = `{` *namedSelectorList* `}`

*namedSelectorList* = *namedSelector* [`,`*namedSelectorList* ]

*namedSelector* = *field* | *name*`:`*expression* | *expression*` as `*name* 

*expression* = *field* | *dlinqExpression*

*field* = Any field of a referenced entity (get from [here](https://plan.tpondemand.com/api/v1/index/meta)) or projected object

*dlinqExpression* = A valid .NET expression with some extensions (see below)

***

Every *objectExpresssion* will be represented as JSON object with given properties calculated from *namedSelectors*. For instance:

<https://plan.tpondemand.com/api/v2/UserStory?select={id,storyName:name,project:{project.id,project.name}}> 

```JSON
{
  "next": "http://plan.tpondemand.com/api/v2/UserStory?where=&select={id,storyName:name,project:{project.id,project.name}}&take=25&skip=25&type=UserStory",
  "items": [
    {
      "id": 93114,
      "storyName": "Study Projectplace tool",
      "project": {
        "id": 37819,
        "name": "AVM: MKT_Backlog"
      }
    },
    {
      "id": 93112,
      "storyName": "Briefly describe api v2",
      "project": {
        "id": 49105,
        "name": "TP3"
      }
    },

```

If a *namedSelector* is a *field* it will get name of the field. In the example above: `id` -> `id`, `project.id` -> `id`, otherwise it will use *name*.

*name* should be a valid alphanumeric identifier: OK - `id`, `camelCase`, `name1`, `name_with_underscore`. Not OK - `name with space`, `1starting_from_number`, `name!with?symbols`.

If you need to get values from inner entities, you need to specify full path to the property: `project.id`, `feature.epic.name`.

For example, I want to get user stories with id, name, effort, project (with id, name and process name), feature (with id, name and epic)

<https://plan.tpondemand.com/api/v2/UserStory?select={id,name,project:{project.id,project.name,process:project.process.name},feature:{feature.id,feature.name,feature.epic}}> 

Let's format selector for convenience:
```
{
   id,
   name,
   project: { 
     project.id,
     project.name,
     process:project.process.name
   },
   feature: {
      feature.id,
      feature.name,
      feature.epic
   }
}
```

The result is:

```Javascript
   { // fully filled userstory
    "id": 93108,
    "name": "Add Left/Right arrows to scroll widgets library",
    "project": {
      "id": 49105,
      "name": "TP3",
      "process": "Kanban TP"
    },
    "feature": {
      "id": 49126,
      "name": "Dashboards MVF",
      "epic": { // as far as epic was requested as a field, it contains only "id" and "name"
        "id": 90678,
        "name": "Dashboards"
      }
    },
    {
      "id": 93114,
      "name": "Study Projectplace tool",
      "project": {
        "id": 37819,
        "name": "AVM: MKT_Backlog",
        "process": "Sales: General"
      },
      "feature": { // This UserStory has no feature, so result projection will include the empty "feature" object
        
      }
    }
```

Along with simple fields and related entities, you also can request inner collections with filters and projections and aggregations for inner collections. E.g. I want to get projects with:
* all User Stories
* all non-closed bugs
* sum of feature efforts
* count of opened requests

<https://plan.tpondemand.com/api/v2/Project?select={id,name,userStories,openBugs:Bugs.Where(EntityState.IsFinal!=true),featureEffort:features.sum(effort),openRequestCount:requests.Count(EntityState.IsInitial==true)}>

* `userstories` - list of all User Stories in format `{id,name}`. As far it is just a field, it doesn't require explicit name
* `openBugs:Bugs.Where(EntityState.IsFinal!=true)` - all non-closed bugs. We need to specify name of the result collection. As far as Bugs is a collection of bug we can use some standard Enumerable methods (see below)
* `featureEffort:features.sum(effort)` - we can aggregate collections by some field of collection item.
* `openRequestCount:requests.Count(EntityState.IsInitial==true)` - we can calculate counts with some filters.

# .NET Expressions

API v2 uses LINQ expression parser that use syntax that is very close to standard .NET syntax. There are some difference:

* Casts are not supported, but you can use [Convert](http://msdn.microsoft.com/en-us/library/system.convert%28v=vs.110%29.aspx) class.
* Constructors are not supported, but you can construct your anonymous objects using syntax described above: `{ex1,ex2}` 
* You can use static methods of some types (DateTime, Convert, Guid). E.g. `DateTime.Now`, `Convert.ToInt`
* You can use Nullable types and checks for nulls 
* You can use instance methods of simple types (e.g., `object.ToString()`, `string.Length`)
* You can use [IIF](http://msdn.microsoft.com/en-us/library/27ydhh0d(v=vs.90).aspx) operator instead of `a?b:c`
* You can use operators (`-`,`+`,`*`,`/`), comparisons (`<`,`>`,`<=`,`>=`,`==`), Boolean constants (`true`,`false`), string constants (`"Some string"`), chars (`'c'`) and `null` identifier.
* You can use operator `in` (`id in [1,2,3]`) 

Also some limited subset of IQueryable methods are supported for nested collections (e.g. bugs of user story):

* `Select(`*selector*`)` - projection as described above
* `Where(`*filter*`)` - filters as described below
* `Sum(`*expression*`)` (as well as Max, Min, Average) - aggregations over specified expression
* `Count(`*filter*`)` - count of filtered items

# Filters

Filters should be valid .NET expressions that returns true or false.

You can use identifier 'it' to reference the parameter of a filter. For example, I want list of all efforts greater than 5:

[{id,name,efforts:**userStories.Select(effort).Where(it>5)**}](https://plan.tpondemand.com/api/v2/Project?select={id,name,efforts:userStories.Select(effort).Where(it%3E5)})

```JSON
{
      "id": 35923,
      "name": "Articles",
      "efforts": [
        10.0000,
        21.0000
      ]
    },
    {
      "id": 70163,
      "name": "Astroport",
      "efforts": [
        
      ]
    },
    {
      "id": 53775,
      "name": "AVM: CFO",
      "efforts": [
        9.5000
      ]
    },
```

# Response format

Response is JSON object with requested collection in `items` field. For instance:

```json
{
  "next": "http://plan.tpondemand.com/api/v2/UserStory?where=&select={id,name,endDate:(endDate-StartDAte).TotalDays}&take=25&skip=25&type=UserStory",
  "items": [
    {
      "id": 93110,
      "name": "Widgets API documentation"
    }
  ]
}
```

All fields are in camelCase, all null values are omitted.

# Custom fields

There are two possibilities to get custom fields from api v2:

* `CustomValues["Custom Field Name"]` - the response will include custom field value only
* `CustomValues.Get("Custom Field Name")` - the response will include custom field value with some info about the field.

For instance, we have Date custom field named DateCF defined for UserStory:

`http://localhost/TargetProcess/api/v2/UserStory?select={id,dateCF:CustomValues.Get("DateCF"),DateCFRaw:CustomValues["DateCF"]}`

Response:
```javascript
{
  "items": [
    {
      "id": 22 // This UserStory has no DateCF custom field defined, so there is no such filed in the returned item
    },
    {
      "id": 11,    // This UserStory has a custom field DateCF, but the field is empty, so the response
      "dateCF": {  // will include only dateCF item with name, type and entityKind, but without a value
        "name": "DateCF",
        "type": "Date",
        "entityKind": "UserStory"
      }
    },
    {
      "id": 8,
      "dateCF": {
        "name": "DateCF",
        "type": "Date",
        "entityKind": "UserStory",
        "value": "\/Date(1415826000000+0300)\/"   // This userStory has DateCF field with value set
      },
      "dateCFRaw": "\/Date(1415826000000+0300)\/"
    }
  ]
}
```
API v2 is tolerate for custom fields - if there is no such custom field defined for the entity, API v2 will not fail, just no return the custom field value.

Custom fields are not supported in filters in general. The only supported filter kind is to check whether the cusotm field is set or not:

`&where=CustomValues["DateCF"]!=null`
`&where=CustomValues["DateCF"]==null`
