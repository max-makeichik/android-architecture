# TODO-MVP-RXJAVA 2.x

Project owners: [Erik Hellman](https://github.com/erikhellman) & [Florina Muntenescu (upday)](https://github.com/florina-muntenescu)

### Summary

This sample is based on the TODO-MVP-RXJAVA project and uses RxJava 2.x and SqlBrite 2.x.

Compared to the TODO-MVP, both the Presenter contracts and the implementation of the Views stay the same. The changes are done to the data model layer and in the implementation of the Presenters. For the sake of simplicity we decided to keep the RxJava usage minimal, leaving optimizations like RxJava caching aside.

The data model layer exposes RxJava ``Observable`` streams as a way of retrieving tasks. The ``TasksDataSource`` interface contains methods like:

```java
Single<List<Task>> getTasks();

Maybe<Task> getTask(@NonNull String taskId);
```

This is implemented in ``TasksLocalDataSource`` with the help of [SqlBrite](https://github.com/square/sqlbrite). The result of queries to the database being easily exposed as streams of data.

```java
@Override
public Single<List<Task>> getTasks() {
    ...
    return mDatabaseHelper.createQuery(TaskEntry.TABLE_NAME, sql)
            .mapToList(mTaskMapperFunction).singleOrError();
}
```

The ``TasksRepository`` combines the streams of data from the local and the remote data sources, exposing it to whoever needs it. In our project, the Presenters and the unit tests are actually the consumers of these ``Observable``s.

The Presenters subscribe to the ``Observable``s from the ``TasksRepository`` and after manipulating the data, they are the ones that decide what the views should display, in the ``.subscribe(...)`` method. Also, the Presenters are the ones that decide on the working threads. For example, in the ``StatisticsPresenter``, we decide on which thread we should do the computation of the active and completed tasks and what should happen when this computation is done: show the statistics, if all is ok; show loading statistics error, if needed; and telling the view that the loading indicator should not be visible anymore.

```java
...
Disposable disposable = Single
        .zip(completedTasks, activeTasks, (completed, active) -> Pair.create(active, completed))
        .subscribeOn(mSchedulerProvider.computation())
        .observeOn(mSchedulerProvider.ui())
        .doAfterTerminate(() -> {
            if (!EspressoIdlingResource.getIdlingResource().isIdleNow()) {
                EspressoIdlingResource.decrement(); // Set app as idle.
            }
            mStatisticsView.setProgressIndicator(false);
        })
        .subscribe(
                // onSuccess
                stats -> mStatisticsView.showStatistics(stats.first.intValue(), stats.second.intValue()),
                // onError
                throwable -> mStatisticsView.showLoadingStatisticsError());
```

Handling of the working threads is done with the help of RxJava's `Scheduler`s. For example, the creation of the database together with all the database queries is happening on the IO thread. The `subscribeOn` and `observeOn` methods are used in the Presenter classes to define that the `Observer`s will operate on the computation thread and that the observing is on the main thread.

### Dependencies

* [RxJava](https://github.com/ReactiveX/RxJava)
* [RxAndroid](https://github.com/ReactiveX/RxAndroid)
* [SqlBrite](https://github.com/square/sqlbrite)

### Java 8 Compatibility

This project uses [lambda expressions](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html) extensively, one of the features of [Java 8](https://developer.android.com/guide/platform/j8-jack.html). To check out how the translation to lambdas was made, check out [this commit](https://github.com/googlesamples/android-architecture/pull/240/commits/929f63e3657be8705679c46c75e2625dc44a5b28), where lambdas and the Jack compiler were enabled.

## Features

### Complexity - understandability

#### Use of architectural frameworks/libraries/tools:

Building an app with RxJava is not trivial as it uses new concepts.

#### Conceptual complexity

Developers need to be familiar with RxJava, which is not trivial.

### Testability

#### Unit testing

Very High. Given that the RxJava ``Observable``s are highly unit testable, unit tests are easy to implement.

#### UI testing

Similar with TODO-MVP

### Code metrics

Compared to TODO-MVP, new classes were added for handing the ``Schedulers`` that provide the working threads.

```
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Java                            49           1110           1413           3740 (3450 in MVP)
XML                             34             97            337            601
-------------------------------------------------------------------------------
SUM:                            83           1207           1750           4341

```
### Maintainability

#### Ease of amending or adding a feature

High.

#### Learning cost

Medium as RxJava is not trivial.
