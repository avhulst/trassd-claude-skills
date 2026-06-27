# Association mapping in full

A `ManyToOne` / `OneToMany` is a single association seen from two sides. The
owning side carries the foreign key and must be set for the relation to persist.

## Owning side (ManyToOne) — required

```php
// src/Entity/Product.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

class Product
{
    #[ORM\ManyToOne(targetEntity: Category::class, inversedBy: 'products')]
    private ?Category $category = null;

    public function getCategory(): ?Category
    {
        return $this->category;
    }

    public function setCategory(?Category $category): self
    {
        $this->category = $category;

        return $this;
    }
}
```

Use a non-nullable join column (`#[ORM\JoinColumn(nullable: false)]` or the
make:entity "nullable: no" answer) when the relation is mandatory.

## Inverse side (OneToMany) — optional

Only add this if you need to navigate from the one side. The collection must
implement Doctrine's `Collection` interface and be initialized to an
`ArrayCollection`.

```php
// src/Entity/Category.php
namespace App\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

class Category
{
    #[ORM\OneToMany(targetEntity: Product::class, mappedBy: 'category')]
    private Collection $products;

    public function __construct()
    {
        $this->products = new ArrayCollection();
    }

    /** @return Collection<int, Product> */
    public function getProducts(): Collection
    {
        return $this->products;
    }

    public function addProduct(Product $product): self
    {
        if (!$this->products->contains($product)) {
            $this->products[] = $product;
            $product->setCategory($this); // sets the OWNING side
        }

        return $this;
    }

    public function removeProduct(Product $product): self
    {
        if ($this->products->contains($product)) {
            $this->products->removeElement($product);
            // null the owning side unless already changed
            if ($product->getCategory() === $this) {
                $product->setCategory(null);
            }
        }

        return $this;
    }
}
```

The key line in `addProduct()` is `$product->setCategory($this)` — updating the
owning side is what makes the change persist. Without the generated sync helpers,
calling `$category->addProduct()` alone would not update the database.

> Iterating/`contains()` on an inverse side with many records can load large
> result sets and cause high memory use. Map the inverse side only when needed.

## Orphan removal

To delete children that are removed from the parent (instead of nulling the FK):

```php
#[ORM\OneToMany(targetEntity: Product::class, mappedBy: 'category', orphanRemoval: true)]
private Collection $products;
```

## Saving and fetching

Set the whole object on the owning side and persist both; Doctrine writes the
foreign key.

```php
$product->setCategory($category);
$entityManager->persist($category);
$entityManager->persist($product);
$entityManager->flush();
```

Related objects are lazily loaded through proxy objects: `$product->getCategory()`
issues a second query only when the data is accessed. Join-fetch (see
repositories.md) to retrieve both in one query and get the real object back.

## ManyToMany

Used when both sides can have many of the other (e.g. students ↔ classes); mapped
with a join table. You choose which side is the owning side. `OneToOne` is mapped
similarly to `ManyToOne` in practice.
