# Basket

**The missing link between your product pages and your payment gateway**

## Money and Currency
Dealing with Money and Currency in an ecommerce application can be fraught with difficulties. Instead of passing around dumb values, we can use Value Objects that are immutable and protect the invariants of the items we hope to represent:
``` php
use Money\Money;
use Money\Currency;

$price = new Money(500, new Currency('GBP'));
```

Equality is important when working with many different types of currency. You shouldn't be able to blindly add two different currencies without some kind of exchange process:
``` php
$money1 = new Money(500, new Currency('GBP'));
$money2 = new Money(500, new Currency('USD'));

// Throws Money\InvalidArgumentException
$money->add($money2);
```

This package uses [mathiasverraes/money](https://github.com/mathiasverraes/money) by [@mathiasverraes](https://github.com/mathiasverraes) throughout to represent Money and Currency values.

## Tax Rates
One of the big problems with dealing with international commerce is the fact that almost everyone has their own rules around tax.

To make tax rates interchangeable we can encapsulate them as objects that implement a common `TaxRate` interface:
``` php
interface TaxRate
{
    /**
     * Return the Tax Rate as a float
     *
     * @return float
     */
    public function float();

    /**
     * Return the Tax Rate as a percentage
     *
     * @return int
     */
    public function percentage();
}
```

An example `UnitedKingdomValueAddedTax` implementation can be found in this package. If you would like to add a tax rate implementation for your country, state or region, please feel free to open a pull request.

## Jurisdictions
Almost every country in the world has a different combination of currency and tax rates. Countries like the USA also have different tax rates within each state.

In order to make it easier to work with currency and tax rate combinations you can think of the combination as an encapsulated "jurisdication". This means you can easily specify the currency and tax rate to be used depending on the location of the current customer.

Jurisdictions should implement the `Jurisdiction` interface:
``` php
interface Jurisdiction
{
    /**
     * Return the Tax Rate
     *
     * @return TaxRate
     */
    public function rate();

    /**
     * Return the currency
     *
     * @return Money\Currency
     */
    public function currency();
}
```

Again, if you would like to add an implementation for your country, state or region, please feel free to open a pull request.

## Products
Each item of the basket is encapsulated as an instance of `Product`. The majority of your interaction with the `Product` class will be through the `Basket`, however it is important that you understand how the `Product` class works.

The `Product` class captures the current state of the each item in the basket. This includes the price, quantity and any discounts that should be applied.

To create a new `Product`, pass the product's [SKU](http://en.wikipedia.org/wiki/Stock_keeping_unit), name, price and tax rate:
``` php
use Money\Money;
use Money\Currency;
use PhilipBrown\Basket\TaxRates\UnitedKingdomValueAddedTax;

$sku   = '1';
$name  = 'Four Steps to the Epiphany';
$rate  = new UnitedKingdomValueAddedTax;
$price = new Money(1000, new Currency('GBP'));
$product = new Product($sku, $name, $price, $rate);
```

The SKU, name, price and tax rate should not be altered once the `Product` is created and so there are no setter methods for these properties on the object.

Each of the `Product` object's `private` properties are available as pseudo `public` properties via the `__get()` magic method:
``` php
$product->sku;    // '1'
$product->name;   // 'Four Steps to the Epiphany'
$product->rate;   // UnitedKingdomValueAddedTax
$product->price;  // Money\Money
```

### Quantity
By default, each `Product` instance will automatically be set with a quantity of 1. You can set the quantity of the product in one of three ways:
``` php
$product->quantity(2);

$product->increment();

$product->decrement();

// Return the `quantity`
$product->quantity;
```

### Freebie
A product can be optionally set as a freebie. This means that the value of the product will not be included during the reconciliation process:
``` php
$product->freebie(true);

// Return the `freebie` status
$product->freebie;
```

By default the `freebie` status of each `Product` is set to `false`.

### Taxable
You can also mark a product as not taxable. By default all products are set to incur tax. By setting the `taxable` status to `false` the taxable value of the product won't be calculated during reconciliation:
``` php
$product->taxable(false);

// Return the `taxable` status
$product->taxable;
```

### Delivery
If you would like to add an additional charge for delivery for the product you can do so by passing an instance of `Money\Money` to the `delivery()` method:
``` php
use Money\Money;
use Money\Currency;

$product->delivery(new Money(500, new Currency('GBP')));

// Return the `delivery` charge
$product->delivery;
```

The `Currency` of the delivery charge must be the same as the `price` that was set when the object was instantiated. By default the delivery charge is set to `0`.

### Coupons
If you would like to record a coupon on the product, you can do so by passing a value to the `coupon()` method:
``` php
$product->coupons('FREE99');

// Return the `coupons` Collection
$product->coupons;
```

You can add as many coupons as you want to each product. The `coupons` class property is an instance of `Collection`. This is an iterable object that allows you to work with an array in an object orientated way.

The coupon itself does not cause the product to set a discount, it is simply a way for you to record that the coupon was applied to the product.

### Tags
Similar to coupons, tags allow you to tag a product so you can record experiments or A/B testing:
``` php
$product->tags('campaign_123456');

// Return the `tags` Collection
$product->tags;
```

The `tags` class property is also an instance of `Collection`.

### Discounts
Discounts are objects that can be applied during the reconciliation process to reduce the price of a product. Each discount object should implement the `Discount` interface:
``` php
interface Discount
{
    /**
     * Calculate the discount on a Product
     *
     * @param Product
     * @return Money\Money
     */
    public function product(Product $product);

    /**
     * Return the rate of the Discount
     *
     * @return mixed
     */
    public function rate();
}
```

There are two discount objects supplied with this package that allow you to set a value discount or a percentage discount:
``` php
PhilipBrown\Basket\Discounts\ValueDiscount;
PhilipBrown\Basket\Discounts\PercentageDiscount;

$product->discount(new PercentageDiscount(20));
$product->discount(new ValueDiscount(new Money(500, new Currency('GBP'))));

// Return the `Discount` instance
$product->discount;
```

### Categories
If you want to apply a set of rules to all products of a certain type, you can define a category object that can be applied to a `Product` instance.

Each category object should implement the `Category` interface:
``` php
interface Category
{
    /**
     * Categorise a Product
     *
     * @param Product $product
     * @return void
     */
    public function categorise(Product $product);
}
```

`PhysicalBook` is an example of a `Category` object that is supplied with this package. When applied to a product, the `PhyisicalBook` will automatically set the `taxable` status to `false`:
``` php
use PhilipBrown\Basket\Categories\PhysicalBook;

$product->category(new PhysicalBook);

// Return the `Category` instance
$product->category;
```

### Actions
Finally if you want to run a series of actions on a product, you can pass a `Closure` to the `action()` method:
``` php
$product->action(function ($product) {
    $product->quantity(3);
    $product->freebie(true);
    $product->taxable(false);
});
```
