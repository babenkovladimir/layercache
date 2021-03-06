# layercache [![CircleCI](https://circleci.com/gh/appmattus/layercache/tree/master.svg?style=svg)](https://circleci.com/gh/appmattus/layercache/tree/master) [![Coverage Status](https://coveralls.io/repos/github/appmattus/layercache/badge.svg?branch=master)](https://coveralls.io/github/appmattus/layercache?branch=master)


Caching made simple for Android and Kotlin.


An important component of building [offline-first architectures](https://developer.android.com/develop/quality-guidelines/building-for-billions-connectivity.html#network-duplicate) is to implement caching.

> An offline-first architecture initially tries to fetch data from local storage
> and, failing that, requests the data from the network. After being retrieved
> from the network, the data is cached locally for future retrieval. This helps
> to ensure that network requests for the same piece of data only occur
> once—with subsequent requests satisfied locally.

![](README/cache-flowchart.png)

At its simplest, a cache is nothing more than a key-value store with a getter
and setter.

```kotlin
interface Cache<Key : Any, Value : Any> {
    fun get(key: Key): Deferred<Value?>
    fun set(key: Key, value: Value): Deferred<Unit>
}
```

The real power comes when we are able to compose two caches into a new cache. A
memory cache should have a single responsibility to store data in memory, and a
disk cache a single responsibility to store data on disk.

```kotlin
val cache = memoryCache.compose(diskCache)
```

For more details read [Caching made simple on Android](https://medium.com/@appmattus/caching-made-simple-on-android-d6e024e3726b)
or watch the talk from [droidcon London 2017](https://skillsmatter.com/skillscasts/11062-lightning-talk-building-composable-caches-on-android).

## Getting started

### Base module

```groovy
compile 'com.appmattus:layercache:<latest-version>'
```

#### Composing two caches

When `get` is called on the composed cache, the first cache will be checked and
if available returned. If not the second cache will be checked and if available
set in the first cache and returned.

```kotlin
composedCache: Cache<Key, Value> = firstCache.compose(secondCache)
```

#### Transforming values

Transform values between data types. This can be used for serialisation and
encryption amongst other things. i.e. `Cache<Key, Value> → Cache<Key, MappedValue>`

For two way transformations:

```kotlin
val cache: Cache<Key, Value> = ...
val valueTransform: Cache<Key, MappedValue> = cache.valueTransform(transform, inverseTransform)

// or

val cache: Cache<Key, Value> = ...
val valueTransform: Cache<Key, MappedValue> = cache.valueTransform(TwoWayTransform)
```

One way transforms return a Fetcher instead of Cache, but otherwise work in the
same way. A Fetcher simply implements no-op for `set` and `evict`.

```kotlin
val fetcher: Fetcher<Key, Value> = ...
val valueTransform: Cache<Key, MappedValue> = cache.valueTransform(transform)

// or

val fetcher: Fetcher<Key, Value> = ...
val valueTransform: Cache<Key, MappedValue> = cache.valueTransform(OneWayTransform)
```

#### Transforming keys

Transform keys to a different data type. i.e. `Cache<Key, Value> → Cache<MappedKey, Value>`

```kotlin
val cache: Cache<Key, Value> = ...
val keyTransform: Cache<MappedKey, Value> = cache.keyTransform(transform)

// or

val cache: Cache<Key, Value> = ...
val keyTransform: Cache<MappedKey, Value> = cache.keyTransform(OneWayTransform)
```

#### Re-using in flight requests

If a get request is already in flight then this ensures the original request is
returned. This may be necessary for disk and network requests along with
transformations that take time to execute.

```kotlin
val newCache: Cache<Key, Value> = cache.reuseInflight()
```

### Retrofit module

```groovy
compile 'com.appmattus:layercache-retrofit:<latest-version>'
```

Given a Retrofit service that returns a Call<Value>, we can turn this into a Cache with `Cache.fromRetrofit`:

```kotlin
interface RetrofitService {
    @GET("get/{key}")
    fun aRequest(@Path("key") key: Key): Call<Value>
}

val service = retrofit.create(RetrofitService::class.java)

...

val cache : Cache<Key, Value> = Cache.fromRetrofit { key: Key ->
    service.aRequest(key)
}
```

### Serializer module

```groovy
compile 'com.appmattus:layercache-serializer:<latest-version>'
```

Configures a transformation from JSON to a serialisable data class and
vice-versa. This is useful to store data on disk or in a database.
i.e. `Cache<Key, String> → Cache<Key, Value>`

```kotlin
@Serialize
data class Value(val value)

val cache: Cache<Key, String> = ...

val objectCache: Cache<Key, Value> = cache.jsonSerializer(Value::class.serializer())

```

### Android base module

```groovy
compile 'com.appmattus:layercache-android:<latest-version>'
```

#### LruCache

```kotlin
val memoryCache: Cache<Key, Value> = Cache.createLruCache(maxSize: Int)

// or

val memoryCache: Cache<Key, Value> = Cache.fromLruCache(...)
```

#### DiskLruCache

```kotlin
val memoryCache: Cache<String, String> = Cache.createDiskLruCache(directory: File, maxSize: Long)

// or

val memoryCache: Cache<String, String> = Cache.fromDiskLruCache(...)
```

### Android LiveData module

```groovy
compile 'com.appmattus:layercache-android-livedata:<latest-version>'
```

Given a cache we can convert it for use with LiveData. This makes the getter
return a `LiveDataResult` which can be one of `Loading`, `Success` or `Failure`.

```kotlin
val liveDataCache = cache.toLiveData()

liveDataCache.get("key").observe(owner) { liveDataResult ->
    when (liveDataResult) {
        is LiveDataResult.Loading -> {
            // display in progress
        }
        is LiveDataResult.Success -> {
            // display liveDataResult.value
        }
        is LiveDataResult.Failure -> {
            // display liveDataResult.exception
        }
    }
}

```

## Download [![Download](https://api.bintray.com/packages/appmattus/maven/layercache/images/download.svg) ](https://bintray.com/appmattus/maven/layercache/_latestVersion)

Available from jcenter()

```groovy
dependencies {
    compile 'com.appmattus:layercache:<latest-version>'

    // To use with the Kotlin serializer
    compile 'com.appmattus:layercache-serializer:<latest-version>'

    // Provides support for ehcache
    compile 'com.appmattus:layercache-ehcache:<latest-version>'

    // Enables converting Retrofit calls to Cache
    compile 'com.appmattus:layercache-retrofit:<latest-version>'

    // Provides LruCache & DiskLruCache support for Android
    compile 'com.appmattus:layercache-android:<latest-version>'

    // Provides one-line String encryption for Android
    compile 'com.appmattus:layercache-android-encryption:<latest-version>'

    // Provides conversion from Cache into LiveData for Android
    compile 'com.appmattus:layercache-android-livedata:<latest-version>'
}
```

## Contributing
Please fork this repository and contribute back using [pull requests](https://github.com/appmattus/layercache/pulls).

All contributions, large or small, major features, bug fixes, additional language translations, unit/integration tests are welcomed.

## License [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

Copyright 2017 Appmattus Limited

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
