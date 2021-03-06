[//]: # (TITLE: Service Decoration in Practice)
[//]: # (DATE: 2017-01-12T08:00:00+01:00)
[//]: # (TAGS: php, doctrine, entity, decoration, symfony, dependency injection container)

[wouterj-repositories-are-just-collections]: http://wouterj.nl/2016/12/repositories-are-just-collections/
[wiki-composition-over-inheritance]: https://en.wikipedia.org/wiki/Composition_over_inheritance
[fig-psr-6-cache]: http://www.php-fig.org/psr/psr-6/
[symfony-service-decoration]: http://symfony.com/doc/current/service_container/service_decoration.html
[symfony-functional-tests]: https://symfony.com/doc/current/testing.html#functional-tests

Recently [WouterJ has written an excelent article about repositories][wouterj-repositories-are-just-collections] and how to treat them as collections. In it he also shows that it is useful to have interfaces on your repository classes. If you have not yet read it, I fully recommend doing so. 

This concept is something I have recently applied in a project and found that there are some extra benefits in using those interfaces. In this post I would like to expand on WouterJ's ideas and show how this enables service decoration.

## The set-up
For the sake of example, I would like to simplify the repository interface to keep the examples small. The repository will only define the `::get()` and `::add()` methods and leave the rest. The interface would look like so:

```php
namespace App\Entity;

interface ProductRepositoryInterface
{
    /**
     * @throws ProductNotFoundException when no product is found for the id
     */
    public function get(int $id): Product;
    public function add(Product $product): void;
}
```
Simple enough. A Doctrine implementation of the interface using a `Product` entity would look like:
```php
namespace App\Entity;

use Doctrine\ORM\EntityManagerInterface;

final class ProductRepository implements ProductRepositoryInterface
{
    private $em;

    public function __construct(EntityManagerInterface $em)
    {
        $this->em = $em;
    }

    public function get(int $id): Product
    {
        if (null === ($product = $this->em->find(Product::class, $id))) {
            throw new ProductNotFoundException();
        }
        
        return $product;
    }
    
    public function add(Product $product): void
    {
        $this->em->persist($product);
    }
}
```
And for the service definition:
```yaml
services:
    app.product_repository:
        class: App\Entity\ProductRepository
        arguments:
            - "@doctrine.orm.entity_manager"
```
Okay, nothing special so far. So why is this useful?

## Extending by decorating

Requirements keep changing as time goes on. What was a good decision now might come to haunt you later on. This is why we tend to stick to best-practices and software patterns. They have proven themselves flexible enough to handle changing situations. One that comes to mind when discussing extending a feature of something is [*Composition over Inheritance*][wiki-composition-over-inheritance].

Decoration is something you see often when discussing Composition. In essence it is creating a wrapper class for an existing implementation of something. Having a common interface can help if you wish to substitute the new implementation for the old. Moreover, most of the time you want to extend rather than replace. So it can even help to contain the original implementation in your new one so you can delegate calls.

For instance, in our example we want to introduce [caching][fig-psr-6-cache]. You can do this in a couple of ways, but using the decorator pattern will provide an easy solution. In that way we can extend the previous class with an implementation that can cache items.

An implementation using the PSR-6 `CacheItemPoolInterface` can look like:
```php
namespace App\Entity;

use Psr\Cache\CacheItemPoolInterface;

final class CachedProductRepository implements ProductRepositoryInterface
{
    private $product_repository;
    private $cache;

    public function __construct(
        ProductRepositoryInterface $product_repository, 
        CacheItemPoolInterface $cache
    ) {
        $this->product_repository = $product_repository;
        $this->cache = $cache;
    }

    public function get(int $id): Product
    {
        $item = $this->cache->getItem((string) $id);

        if (!$item->isHit()) {
            $product = $this->product_repository->get($id);

            $item->set($product);
            $this->cache->save($item);
        }

        return $item->get();
    }
    
    public function add(Product $product): void
    {
        $this->product_repository->add($product);
    }
}
```
We can then [decorate the original services][symfony-service-decoration] with the cached version.
```yaml
services:
    app.product_repository.cached:
        class: App\Entity\CachedProductRepository
        public: false
        decorates: app.product_repository
        arguments:
            - "@app.product_repository.cached.inner"
            - "@cache.app"
```
And done! The repository is now using caching to load products. No changes had to made to the old code or even the old services definitions. All we did was add.

As seen in the example, the `CachedProductRepository` still provides all the functionality of the Doctrine `ProductRepository`. Moreover, it does not duplicate the code but reuses the `ProductRepository` to do so. That also means that responsibilities of these classes are well defined. One is interacting with doctrine; the other is providing caching on top if it. 

These clear responsibilities also have another added benefit. Clear responsibilities is usually a good sign that it can be unit tested with ease. And speaking of testing, what about functional testing?

## Even more decoration

At some point you decide you want to start functional testing your applications. [Symfony provides the `WebTestCase`][symfony-functional-tests] which can be used for functional testing. However, all those caches and database dependencies are hard to work around. A solution would be to create an array implementation of the `ProductRepositoryInterface` and use that for testing.

An implementation can look like:
```php
namespace App\Entity\Test;

use App\Entity\Product;
use App\Entity\ProductNotFoundException;
use App\Entity\ProductRepositoryInterface;

final class ArrayProductRepository implements ProductRepositoryInterface
{
    private $products = [];
    
    public function get(int $id): Product
    {
        foreach ($this->products as $product) {
            if ($product->getId() === $id) {
                return $product;
            }
        }
        throw new ProductNotFoundException();
    }
    
    public function add(Product $product): void
    {
        $this->products[] = $product;
    }
}
```
And in the `config_test.yml` you can decorate the original services.
```yaml
services:
    app.product_repository.array:
        class: App\Entity\Test\ArrayProductRepository
        public: false
        decorates: app.product_repository
```

Now the functional test no longer depends on doctrine nor the cache. This should make testing a lot easier and moreover, a `Product` can easily be inserted for your tests. This allows cleaner tests and less bootstrapping.

## In conclusion

Decorating services in Symfony sometimes seems like a strange concept and not that useful. With this post I have tried to show a more real-world application of decorating in Symfony. Moreover, when combined with good interfaces it can be a powerful feature to use.
