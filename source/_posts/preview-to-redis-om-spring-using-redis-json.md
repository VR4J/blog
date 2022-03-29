---
title: Preview to Redis OM Spring using RedisJson
date: 2022-03-29 11:30:00
icon: 🔥
image: /images/preview-to-redis-om-spring-using-redis-json-social.jpg
tags:
    - Java 17
    - Spring Boot
    - Redis
    - Redis OM
    - RedisJson
excerpt: After the announcement made last week about RedisJson 2.0 in RedisDays London 2022, lets have a look at how to use it in Spring Boot!
---

Just a few days ago, **Redis CEO Ofer Bengal** announced the global availability of [RedisJson 2.0](https://redis.com/press/redisjson-2-0-serves-as-fast-flexible-document-database/). Wondering why that is all cool and worth your time? Well scroll a bit further down and I'll tell ya!

As always, feel free to skip ahead and check out the [Github Repository](https://github.com/VR4J/redis-json-stock-management) containing all the code.

## What is RedisJson
RedisJson is an in-memory document store that provides real-time access to and processing of JSON documents. RedisJson 2.0 adds native indexing, querying, and full-text search capabilities, all leveraging RediSearch which makes it a very powerfull tool for us Computer Geeks. While document-based databases are becoming a trend because of their flexibility in dynamic scheming, they are still struggling with the performance requirements a relational database offers.

RedisJson seeks these gaps where a document-based database fails to meet the performance requirement, but is perfect for its use-case because of its dynamic scheming.

## What is Redis OM Spring
You're probably wondering.. well how do we use it then? Well it's true that when you look at the latest version of the [Spring Data Redis](https://github.com/spring-projects/spring-data-redis) library it will just use @RedisHash annotations not leveraging any of the features mentioned above. Which is why you need to add the [Redis OM Spring](https://github.com/redis/redis-om-spring) library to add all these nice features. **Redis Object Mapping**, or Redis OM for short, provides annotations like `@Document` to map Spring Data models to Redis JSON documents.

## Example
Alright so let's dive into a use-case and see how it works!

Assume we have a webstore that sells all different kinds of things: books, games, good old fashion dvd's, and more. Everything is going easy peasy, but with the rising amount of orders coming in we start to retroactively cancel, or delay orders because items went out of stock faster than we could update our webstore.

Since we live in this hyper modern world we decide to have a look at live stock updates, which need to be fast, reliable but also dynamic to our articles. 

### Setup

```java
@Data
@Document // Redis OM Spring 🎉
public class StockItem<T> {
    @Id
    String id;
    Category category;

    T details;

    double price;
    long stock;

    public enum Category {
        BOOK, GAME
    }
}
```

Every item that we have in our store comes down to being in a certain **category**, having a **price**, and the amount of **stock** that is left. Dependend on the category we can have some specific implementation of details like you can see in the `StockItem` above. 

We can then define the details specific implementations per category like the examples below.

```java
@Data
public class Book {
    String ean;
    String title;
    String author;
    Date published;
}
```

```java
@Data
public class Game {
    String title;
    Platform platform;
    Date released;

    enum Platform {
        PC, PS4, PS5, XBOX_ONE
    }
}
```

Alright! So in order to use and save our RedisJson documents we will need to specify our Repository.

```java
import com.redis.om.spring.repository.RedisDocumentRepository;

public interface StockRepository extends RedisDocumentRepository<StockItem<?>, String> { }
```

**Important**: 
Redis OM Spring will only pickup the repository if you add the `@EnableRedisDocumentRepositories` annotation, however as of today, you also need to specify the `basePackages` property in order for it to be picked up. 

```java
@SpringBootApplication
@EnableRedisDocumentRepositories(basePackages = "nl.vreijsenj.redisjson.stock")
public class StockManagementApplication {
	public static void main(String[] args) {
		SpringApplication.run(StockManagementApplication.class, args);
	}
}
```

### Seeding
We can't start updating the stock without having actual StockItems in Redis, so we create an event listener that seeds Redis on startup using our Redis Repository.

```java
@Component
@RequiredArgsConstructor
public class CatalogSeeder {

    private final StockRepository repository;

    @EventListener
    public void seed(ContextRefreshedEvent event) {
        // 2016-05-10
        Instant released = Instant.ofEpochSecond(1462838400);
        Game uncharted = new Game("Uncharted 4: A Thief's End", Game.Platform.PS4, Date.from(released));

        // 2015-07-01
        Instant published = Instant.ofEpochSecond(1435708800);
        Book dune = new Book("9780340960196", "Dune", "Hubert, Frank", Date.from(published));

        StockItem<Game> game = new CatalogItem<>(null, StockItem.Category.GAME, uncharted, 21.74, 10);
        StockItem<Book> book = new CatalogItem<>(null, StockItem.Category.BOOK, dune, 11.99, 10);

        repository.save(game);
        repository.save(book);
    }
}
```

When we start our application and have a look in redis we can now see we have two StockItems!

```sh
$ redis-cli -h localhost -p 6379 --raw
redis > KEYS *
nl.vreijsenj.redisjson.stock.StockItem
nl.vreijsenj.redisjson.stock.StockItem:01FZ02YKEZ3QM96ZNTJYHSJHB0
nl.vreijsenj.redisjson.stock.StockItem:01FZ02YKCG7XP75A7GGD0YN6DM
redis > JSON.GET nl.vreijsenj.redisjson.stock.StockItem:01FZ02YKEZ3QM96ZNTJYHSJHB0 NEWLINE "\n" SPACE " " INDENT "\t"
{
	"id": "01FZ02YKEZ3QM96ZNTJYHSJHB0",
	"category": "BOOK",
	"details": {
		"ean": "9780340960196",
		"title": "Dune",
		"author": "Hubert, Frank",
		"published": 1435708800000
	},
	"price": 11.99,
	"stock": 10
}
redis > JSON.GET nl.vreijsenj.redisjson.stock.StockItem:01FZ02YKCG7XP75A7GGD0YN6DM NEWLINE "\n" SPACE " " INDENT "\t"
{
	"id": "01FZ02YKCG7XP75A7GGD0YN6DM",
	"category": "GAME",
	"details": {
		"title": "Uncharted 4: A Thief's End",
		"platform": "PS4",
		"released": 1462838400000
	},
	"price": 21.74,
	"stock": 10
}
```

### Updating
We have our data setup, and we can now start updating our stock whenever a new order is placed.

Unfortunately the Redis OM Spring library doesn't (yet) allow partial document updates using the Repository, or a dedicated template bean. Fortunately it does allow us to autowire a `JSONOperations<?>` bean that is capable of doing partial document updates.

Let's say we have an order coming in, and we need to update the stock right away.

```sh
POST /api/v1.0/orders
{
    "user": {
        "forename": "Joery",
        "surname": "Vreijsen"
    },
    "lines": [{
        "stockItemId": "01FZ0APM02XF8NAHTXW6PXTYAC",
        "count": 2
    }]
}
```

We have an `OrderService` which will process all incoming orders, and pick every item on the order from the stock.

```java
@Component
@RequiredArgsConstructor
public class OrderService {

    private final StockService stock;

    public Order process(Order order) {
        order.getLines().forEach(line ->
            stock.pick(line.getStockItemId(), line.getCount())
        );

        return order;
    }
}
```

When picking from the Stock, we call the responsible `StockService`.

```java
@Component
@RequiredArgsConstructor
public class StockService {

    private final JSONOperations<String> ops;

    public synchronized void pick(String id, long count) {
        Long current = ops.get(getKey(id), long.class, Path.of("stock"));

        assert current != null;

        if(current < count) {
            throw new OutOfStockException(
                    String.format("Cannot pick item [%s] as there is not enough stock (%s) available for the requested amount of %s.", id, current, count)
            );
        }

        ops.set(getKey(id), current - count, Path.of("stock"));
    }

    private String getKey(String id) {
        return String.format("nl.vreijsenj.redisjson.stock.StockItem:%s", id);
    }
}
```

The `StockService` will use the `JSONOperations<?>` bean to fetch the stock property from the StockItem and reduce it with the amount that was ordered.

Unfortunately RedisJson does not offer a way (yet) of reducing a numeric property in a single operation, which means theoretically there could be enough stock when we call the `get` operation, when while updating the stock another (outside) process could have already claimed it.

Since in this case we are the only running instance, and we are the only one updating these documents, we can manage by declaring the pick method as `synchronized` to make sure there is no other thread interfering.

## Result

Using the `monitor` command in the `redis-cli` we can monitor every command that is being executed on our redis-server and see whether our stock is being correctly updated.

```sh
1648204926.982903 [0 xxx.xx.x.x:61670] "SISMEMBER" "nl.vreijsenj.redisjson.stock.StockItem" "01FZ0APKZ9SW1MAPXCPCY3HHK6"
1648204926.990580 [0 xxx.xx.x.x:61670] "EXISTS" "nl.vreijsenj.redisjson.stock.StockItem:01FZ0APKZ9SW1MAPXCPCY3HHK6"
1648204926.998838 [0 xxx.xx.x.x:61670] "JSON.SET" "nl.vreijsenj.redisjson.catstockalog.StockItem:01FZ0APKZ9SW1MAPXCPCY3HHK6" "." "{\"id\":\"01FZ0APKZ9SW1MAPXCPCY3HHK6\",\"category\":\"GAME\",\"details\":{\"title\":\"Uncharted 4: A Thief\\u0027s End\",\"platform\":\"PS4\",\"released\":1462838400000},\"price\":21.74,\"stock\":10}"
1648204927.001772 [0 xxx.xx.x.x:61670] "SADD" "nl.vreijsenj.redisjson.stock.StockItem" "01FZ0APKZ9SW1MAPXCPCY3HHK6"
1648204927.005298 [0 xxx.xx.x.x:61670] "SISMEMBER" "nl.vreijsenj.redisjson.stock.StockItem" "01FZ0APM02XF8NAHTXW6PXTYAC"
1648204927.008958 [0 xxx.xx.x.x:61670] "EXISTS" "nl.vreijsenj.redisjson.stock.StockItem:01FZ0APM02XF8NAHTXW6PXTYAC"
1648204927.010938 [0 xxx.xx.x.x:61670] "JSON.SET" "nl.vreijsenj.redisjson.stock.StockItem:01FZ0APM02XF8NAHTXW6PXTYAC" "." "{\"id\":\"01FZ0APM02XF8NAHTXW6PXTYAC\",\"category\":\"BOOK\",\"details\":{\"ean\":\"9780340960196\",\"title\":\"Dune\",\"author\":\"Hubert, Frank\",\"published\":1435708800000},\"price\":11.99,\"stock\":10}"
1648204927.013049 [0 xxx.xx.x.x:61670] "SADD" "nl.vreijsenj.redisjson.stock.StockItem" "01FZ0APM02XF8NAHTXW6PXTYAC"
```

We can see that our `CatalogSeeder` creates the initial StockItems, and keeps an index on the id property.

```sh
1648204941.548383 [0 xxx.xx.x.x:61670] "JSON.GET" "nl.vreijsenj.redisjson.stock.StockItem:01FZ0APM02XF8NAHTXW6PXTYAC" "stock"
1648204941.552428 [0 xxx.xx.x.x:61670] "JSON.SET" "nl.vreijsenj.redisjson.stock.StockItem:01FZ0APM02XF8NAHTXW6PXTYAC" "stock" "8"
```

After that when placing the order, we can see that the `JSON.GET` and the `JSON.SET` operations are called to update the stock in just 5 miliseconds. 
Considering this was run from my humble local environment, I say that is pretty damn fast.

That's all folks! As always if you want to see the full implementation, or just sniff around the codebase, here is the [Github Repository](https://github.com/VR4J/redis-json-stock-management).

Cheerios!