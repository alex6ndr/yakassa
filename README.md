# Пакет для работы с Яндекс.Кассой.

===

Возможности:

* создание платёжной формы;
* передача данных согласно 54-ФЗ;
* обработка уведомлений `checkOrder` и `paymentAviso`.

===

Требования:

* php >= 7.1
* Laravel >= 5.3 

===

# Установка

```bash
composer require appwilio/yakassa
```

Подключение сервис-провайдера:

```php
// config/app.php
'providers' => [
    ...
    Appwilio\YaKassa\YaKassaServiceProvider::class,
],
```

Настройки:

> [Описание параметров](https://tech.yandex.ru/money/doc/payment-solution/shop-config/parameters-docpage/).
```php
// config/services.php
...
    'yakassa' => [
        'test_mode'     => env('YAKASSA_TEST_MODE', true),
        'shop_id'       => env('YAKASSA_SHOP_ID'),
        'showcase_id'   => env('YAKASSA_SHOWCASE_ID'),
        'shop_password' => env('YAKASSA_SHOP_PASSWORD'),        
    ],
...
```
## Подготовка основных данных для платёжной формы

Заказ должен имплементировать интерфейс `\Appwilio\YaKassa\Contracts\YaKassaOrder`:

```php
<?php

use Appwilio\YaKassa\Contracts\YaKassaOrder;

class Order implements YaKassaOrder
{
    public function getOrderSum(): float
    {
        return $this->total;
    }
    
    public function getCustomerNumber(): string
    {
        return $this->customer->id;
    }
    
    public function getOrderNumber(): ?string
    {
        return $this->id;
    }
    
    public function getPaymentType(): ?string
    {
        return 'PC';
    }
    
    public function getCustomerEmail(): ?string
    {
        return $this->customer->email;
    }
    
    public function getCustomerPhone(): ?string
    {
        return $this->customer->phone;
    }
}
```

### Дополнительные данные согласно требованиям 54-ФЗ

> Внимание! Протокол дополняется, текущая версия 2.1. [Общая информация](https://kassa.yandex.ru/blog/fz54-developers), [описание изменений](https://kassa.yandex.ru/docs/API_Yandex.Kassa_54FZ_changes.pdf).


Заказ должен имплементировать интерфейс `\Appwilio\YaKassa\Contracts\YaKassaOrder54FZ`:

```php
<?php

use Appwilio\YaKassa\Contracts\YaKassaOrder54FZ;

class Order implements YaKassaOrder54FZ
{
    public function getOrderSum(): float
    {
        return $this->total;
    }
    
    public function getCustomerNumber(): string
    {
        return $this->customer->id;
    }
    
    public function getItems(): iterable
    {
        return $this->items; // товары/услуги в заказе
    }
    
    public function getTaxSystem(): ?int
    {
        return YaKassaOrder54FZ::TAX_OSN;
    }
    
    public function getCustomerContact(): string
    {
        return $this->customer->phone;
    }
        
    public function getOrderNumber(): ?string
    {
        return $this->id;
    }
    
    public function getPaymentType(): ?string
    {
        return 'PC';
    }
    
    public function getCustomerEmail(): ?string
    {
        return $this->customer->email;
    }
    
    public function getCustomerPhone(): ?string
    {
        return $this->customer->phone;
    }
}
```

Каждая позиция заказа должна имплементировать интерфейс `\Appwilio\YaKassa\Contracts\YaKassaOrderItem54FZ`:

```php
<?php

use Appwilio\YaKassa\Contracts\YaKassaOrderItem54FZ;

class OrderItem implements YaKassaOrderItem54FZ
{
    public function getAmount(): float
    {
        return $this->amount;
    }

    public function getQuantity(): float
    {
        return $this->quantity;
    }

    public function getTaxRate(): int
    {
        return YaKassaOrderItem54FZ::VAT_18;
    }

    public function getCurrency(): ?string
    {
        return null; // равнозначно RUB
    }

    public function getTitle(): string
    {
        return $this->product->title;
    }
}
```

## Создание платёжной формы

В контролллере:

```php
<?php

class OrdersController
{
    public function showPaymentForm(\Appwilio\YaKassa\YaKassa $kassa, $orderId)
    {
        $order = Order::find($orderId);
        
        $paymentForm = $kassa->buildPaymentForm($order);
        
        return view('payment', ['form' => $paymentForm]);
    }    
}

```

В шаблоне:

```blade
    <form method="POST" action="{{ $form->getPaymentUrl() }}">
        @foreach ($form->toArray() as $k => $v) 
            <input type="hidden" name="{{ $k }}" value="{{ $v }}" />
        @endforeach
        
        ...
    </form>
```