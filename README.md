## Dynamic Data

Dynamic Data is a portable class library which brings the power of Reactive Extensions (Rx) to collections.  

Mutable collections frequently experience additions, updates, and removals (among other changes). Dynamic Data provides two collection implementations, `ISourceCache<T,K>` and `ISourceList<T>`, that expose changes to the collection via an observable change set. The resulting observable change sets can be manipulated and transformed using Dynamic Data's robust and powerful array of change set operators. These operators receive change notifications, apply some logic, and subsequently provide their own change notifications. Because of this, operators are fully composable and can be chained together to perform powerful and very complicated operations while maintaining simple, fluent code.

Using Dynamic Data's collections and change set operators make in-memory data management extremely easy and can reduce the size and complexity of your code base by abstracting complicated and often repetitive operations.

###Some links

- [![Join the chat at https://gitter.im/RolandPheasant/DynamicData](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/RolandPheasant/DynamicData?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
- [![Downloads](https://img.shields.io/nuget/dt/DynamicData.svg)](http://www.nuget.org/packages/DynamicData/)	
- [![Build status]( https://ci.appveyor.com/api/projects/status/occtlji3iwinami5/branch/develop?svg=true)](https://ci.appveyor.com/project/RolandPheasant/DynamicData/branch/develop)
- Sample wpf project https://github.com/RolandPheasant/Dynamic.Trader
- Blog at  http://dynamic-data.org/
- You can contact me on twitter  [@RolandPheasant](https://twitter.com/RolandPheasant) or email at [roland@dynamic-data.org]

### Version 4 has been released

The core of Dynamic Data is the observable cache, which is great in most circumstances. However, sometimes the simplicity of an observable list is needed and version 4 delivers this. The addition of the observable list has been a great effort and has consumed loads of my time, mental capacity, and resolve. In spite of the difficulty, the observable list is finally crystallising into a stable state. 

Downloading the latest release of Dynamic Data from [Dynamic Data on nuget](https://www.nuget.org/packages/DynamicData/) will allow you to create and have fun with observable lists.

## Create Dynamic Data Collections

### The Observable List

Create an observable list like this:
```cs
var myInts = new SourceList<int>();
```
The observable list provides the direct edit methods you would expect. For example:
```cs
myInts.AddRange(Enumerable.Range(0, 10000)); 
myInts.Add(99999); 
myInts.Remove(99999);
```
Each of the amendments caused by the AddRange operation above will produce a unique change notification. When making multiple modifications, batch editing a list using the `.Edit` operator is much more efficient and produces only a single change notification.
```cs
myInts.Edit(innerList =>
{
   innerList.Clear();
   innerList.AddRange(Enumerable.Range(0, 10000));
});
```
If ``myInts`` is to be exposed publicly it can be made read only using `.AsObservableList`
```cs
IObservableList<int> readonlyInts = myInts.AsObservableList();
```
which hides the edit methods.

The list's changes can be observed by calling `myInts.Connect()` like this:
```cs
IObservable<IChangeSet<int>> myIntsObservable = myInts.Connect();
```
This creates an observable change set for which there are dozens of operators. The changes are transmitted as an Rx observable, so they are fluent and composable.

### The observable cache

Create an observable cache like this:
```cs
var myCache = new SourceCache<TObject,TKey>(t => key);
```
There are direct edit methods, for example

```cs
myCache.Clear();
myCache.AddOrUpdate(myItems);
```
Each amend or update caused by the AddOrUpdate operation above will produce a unique change notification. When making multiple modifications, batch editing a cache using the `.BatchUpdate` operator is much more efficient and produces only a single change notification.

```cs
myCache.BatchUpdate(innerCache =>
			  {
			      innerCache.Clear();
			      innerCache.AddOrUpdate(myItems);
			  });
```
If `myCache` is to be exposed publicly it can be made read only `.AsObservableCache`

```cs
IObservableCache<TObject,TKey> readonlyCache = myCache.AsObservableCache();
```
which hides the edit methods.

The cache is observed by calling `myCache.Connect()` like this:
```cs
IObservable<IChangeSet<TObject,TKey>> myCacheObservable = myCache.Connect();
```
This creates an observable change set for which there are dozens of operators. The changes are transmitted as an Rx observable, so they are fluent and composable.

## Creating Observable Change Sets
As stated in the introduction of this document, Dynamic Data is based on the concept of creating and manipulating observable change sets. 

The primary method of creating observable change sets is to connect to instances of `ISourceCache<T,K>` and `ISourceList<T>`. There are alternative methods to produce observables change sets however, depending on the data source.

### Connect to a Cache or List
Calling `Connect()` on a `ISourceList<T>` or `ISourceCache<T,K>` will produce an observable change set. 
```cs
var myObservableChangeSet = myDynamicDataSource.Connect();
```

### Create an Observable Change Set from an Rx Observable
Given either of the following observables:
```cs
IObservable<T> myObservable;
IObservable<IEnumerable<T>> myObservable;
```
an observable change set can be created like by calling `.ToObservableChangeSet` like this:
```cs
var myObservableChangeSet = myObservable.ToObservableChangeSet(t=> t.key);
```

### Create an Observable Change Set from an Rx Observable with an Expiring Cache
The problem with the example above is that the internal backing cache of the observable change set will grow in size forever. 
To counter this behavior, there are overloads of `.ToObservableChangeSet` where a size limitation or expiry time can be specified for the internal cache.

To create a time expiring cache, call `.ToObservableChangeSet` and specify the expiry time using the expireAfter argument:
```cs
var myConnection = myObservable.ToObservableChangeSet(t=> t.key, expireAfter: item => TimeSpan.FromHours(1));
```

To create a size limited cache, call `.ToObservableChangeSet` and specify the size limit using the limitSizeTo argument:
```cs
var myConnection = myObservable.ToObservableChangeSet(t=> t.key, limitSizeTo:10000);
```
There is also an overload to specify expiration by both time and size.

### Create an Observable Change Set from an Observable Collection
```cs
var myObservableCollection = new ObservableCollection<T>();
```
To create a cache based observable change set, call `.ToObservableChangeSet` and specify a key selector for the backing cache
```cs
var myConnection = myObservableCollection.ToObservableChangeSet(t => t.Key);
```
or to create a list based observable change set call `.ToObservableChangeSet` with no arguments
```cs
var myConnection = myObservableCollection.ToObservableChangeSet();
```
This method is only recommended for simple queries which act only on the UI thread as `ObservableCollection` is not thread safe.

## Consuming Observable Change Sets
The examples below illustrate the kind of things you can achieve after creating an observable change set. 
No you can create an observable cache or an observable list, here are a few quick fire examples to illustrated the diverse range of things you can do. In all of these examples the resulting sequences always exactly reflect the items is the cache i.e. adds, updates and removes are always propagated.

#### Bind a Complex Stream to an Observable Collection
This example first filters a stream of trades to select only live trades, then creates a proxy for each live trade, and finally orders the results by most recent first. The resulting trade proxies are bound on the dispatcher thread to the specified observable collection. 
```cs
var list = new ObservableCollectionExtended<TradeProxy>();

var myTradeCache = new SourceCache<Trade, long>(trade => trade.Id);

var myTradeCacheObservable = myTradeCache.Connect();

var myOperation = myTradeCacheObservable 
					.Filter(trade=>trade.Status == TradeStatus.Live) 
					.Transform(trade => new TradeProxy(trade))
					.Sort(SortExpressionComparer<TradeProxy>.Descending(t => t.Timestamp))
					.ObserveOnDispatcher()
					.Bind(list) 
					.DisposeMany()
					.Subscribe()
```
Since the TradeProxy object is disposable, the `DisposeMany` operator ensures that the proxies objects are disposed when they are no longer part of this observable stream.

Note that `ObservableCollectionExtended<T>` is provided by Dynamic Data and is more efficient than the standard `ObservableCollection<T>`.

#### Create a Derived List or Cache
This example shows how you can create derived collections from an observable change set. It applies a filter to a collection, and then creates a new observable collection that only contains items from the original collection that pass the filter.
This pattern is incredibly useful when you want make modifications to an existing collection and then expose the modified collection to consumers. 

Even though the code in this example is very simple, this is one of the most powerful aspects of Dynamic Data. 

Given a SourceList 
```cs
var myList = new SourceList<People>();
```
You can apply operators, in this case the `Filter()` operator, and then create a new observable list with `AsObservableList()`
```cs
var oldPeople = myList.Connect().Filter(person => person.Age > 65).AsObservableList();
```
The resulting observable list, oldPeople, will only contain people who are older than 65.

The same pattern can be used with SourceCache by using `.AsObservableCache()` to create derived caches.

#### Filtering
Filter the observable change set by using the `Filter` operator
```cs
var myPeople = new SourceList<People>();
var myPeopleObservable = myPeople.Connect();

var myFilteredObservable = myPeopleObservable.Filter(person => person.Age > 50); 
```
or to filter a change set dynamically 
```cs
IObservable<Func<Person,bool>> observablePredicate=...;
var myFilteredObservable = myPeopleObservable.Filter(observablePredicate); 
```

#### Sorting
Sort the observable change set by using the `Sort` operator
```cs
var myPeople = new SourceList<People>();
var myPeopleObservable = myPeople.Connect();
var mySortedObservable = myPeopleObservable.Sort(SortExpressionComparer.Ascending(p => p.Age); 
```
or to dynamically change sorting
```cs
IObservable<IComparer<Person>> observableComparer=...;
var mySortedObservable = myPeopleObservable.Sort(observableComparer);
```
#### Grouping
The `GroupOn` operator pre-caches the specified groups according to the group selector.
```cs
var myOperation = personChangeSet.GroupOn(person => person.Status)
```

#### Transformation
The `Transform` operator allows you to map objects from the observable change set to another object
```cs
var myPeople = new SourceList<People>();
var myPeopleObservable = myPeople.Connect();
var myTransformedObservable = myPeopleObservable.Transform(person => new PersonProxy(person));
```

The `TransformToTree` operator allows you to create a fully formed reactive tree
```cs
var myPeople = new SourceList<People>();
var myPeopleObservable = myPeople.Connect();
var myTransformedObservable = myPeopleObservable.TransformToTree(person => person.BossId);
```

Flatten a child enumerable
```cs
var myOperation = personChangeSet.TransformMany(person => person.Children) 
```

#### Aggregation
The `Count`, `Max`, `Min`, `Avg`, and `StdDev` operators allow you to perform aggregate functions on observable change sets
```cs
var myPeople = new SourceList<People>();
var myPeopleObservable = myPeople.Connect();

var countObservable = 	 myPeopleObservable.Count();
var maxObservable = 	 myPeopleObservable.Max(p => p.Age);
var minObservable = 	 myPeopleObservable.Min(p => p.Age);
var stdDevObservable =   myPeopleObservable.StdDev(p => p.Age);
var avgObservable = 	 myPeopleObservable.Avg(p => p.Age);
```
More aggregating operators will be added soon.

#### Logical Operators
The `And`, `Or`, `Xor` and `Except` operators allow you to perform logical operations on observable change sets
```cs
var peopleA = new SourceCache<Person,string>(p => p.Name);
var peopleB = new SourceCache<Person,string>(p => p.Name);

var observableA = peopleA.Connect();
var observableB = peopleB.Connect();

var inBoth = observableA.And(observableB);
var inEither= observableA.Or(observableB);
var inOnlyOne= observableA.Xor(observableB);
var inAandNotinB = observableA.Except(observableB);
```
Currently the join operators are only implemented for observable change sets based on `SourceCache`

#### Disposal
The `DisposeMany` operator ensures that objects are disposed when removed from an observable stream
```cs
var myPeople = new SourceList<People>();
var myPeopleObservable = myPeople.Connect();
var myTransformedObservable = myPeopleObservable.Transform(person => new DisposablePersonProxy(person))
                                                .DisposeMany();
```
The `DisposeMany` operator is typically used when a transform function creates disposable objects.

#### Distinct Values
The `DistinctValues` operator will select distinct values from the underlying collection
```cs
var myPeople = new SourceList<People>();
var myPeopleObservable = myPeople.Connect();
var myDistinctObservable = myPeopleObservable.DistinctValues(person => person.Age);
```

#### Virtualisation

Visualise data to restrict by index and segment size
```cs
IObservable<IVirtualRequest> request; //request stream
var virtualisedStream = someDynamicDataSource.Virtualise(request)
```
Visualise data to restrict by index and page size
```cs
IObservable<IPageRequest> request; //request stream
var pagedStream = someDynamicDataSource.Page(request)
```
In either of the above, the result is re-evaluated when the request stream changes

Top is an overload of ```Virtualise()``` and will return items matching the first 'n'  items.
```cs
var topStream = someDynamicDataSource.Top(10)
```
#### Observing binding changes

If the collection has objects which implement ```INotifyPropertyChanged``` the the following operators are available
```cs
var ageChanged = peopleDataSource.WhenValueChanged(p => p.Age)
```
which returns an observable of the age when the value of Age has changes, .
```cs
var ageChanged = peopleDataSource.WhenPropertyChanged(p => p.Age)
```
which returns an observable of the person and age when the value of Age has changes, .
```cs
var personChanged = peopleDataSource.WhenAnyPropertyChanged()
```
which returns an observable of the person when any property has changed,.

#### Observing item changes

Binding is a very small part of Dynamic Data. The above notify property changed overloads are just an example when binding. If you have a domain object which has children observables you can use ```MergeMany()``` which subscribes to and unsubscribes from items according to collection changes.

```cs
var myoperation = somedynamicdatasource.Connect() 
			.MergeMany(trade => trade.SomeObservable());
```
This wires and unwires ```SomeObservable``` as the collection changes.

## History of Dynamic Data
Even before Rx existed I had implemented a similar concept using old fashioned events but the code was very ugly and my implementation full of race conditions so it never existed outside of my own private sphere. My second attempt was a similar implementation to the first but using Rx when it first came out. This also failed as my understanding of Rx was flawed and limited and my design forced consumers to implement interfaces.  Then finally I got my design head on and in 2011-ish I started writing what has become dynamic data. No inheritance, no interfaces, just the ability to plug in and use it as you please.  All along I meant to open source it but having so utterly failed on my first 2 attempts I decided to wait until the exact design had settled down. The wait lasted longer than I expected and end up taking over 2 years but the benefit is it has been trialled for 2 years on a very busy high volume low latency trading system which has seriously complicated data management. And what's more that system has gathered a load of attention for how slick and cool and reliable it is both from the user and IT point of view. So I present this library with the confidence of it being tried, tested, optimised and mature. I hope it can make your life easier like it has done for me.

## Want to know more?
I could go on endlessly but this is not the place for full documentation.  I promise this will come but for now I suggest downloading my WPF sample app (links at top of document)  as I intend it to be a 'living document' and I promise it will be continually maintained. 

Also if you following me on Twitter you will find out when new samples or blog posts have been updated.

Additionally if you have read up to here and not pressed star then why not? Ha. A star may make me be more responsive to any requests or queries.

