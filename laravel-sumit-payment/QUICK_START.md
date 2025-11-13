# Quick Start Guide

Get started with SUMIT Payment Gateway for Laravel in 5 minutes!

## Prerequisites

- Laravel 11 or 12 application
- PHP 8.1 or higher
- SUMIT merchant account
- Composer

## Installation (3 steps)

### Step 1: Install Package

```bash
composer require sumit/laravel-payment-gateway
```

### Step 2: Publish & Migrate

```bash
php artisan vendor:publish --tag=sumit-payment-config
php artisan vendor:publish --tag=sumit-payment-migrations
php artisan migrate
```

### Step 3: Configure

Add to your `.env`:

```env
SUMIT_COMPANY_ID=your-company-id
SUMIT_API_KEY=your-api-key
SUMIT_API_PUBLIC_KEY=your-public-key
SUMIT_MERCHANT_NUMBER=your-merchant-number
```

## Basic Usage

### Simple Payment

```php
use Sumit\LaravelPayment\Facades\SumitPayment;

$result = SumitPayment::processPayment([
    'amount' => 100.00,
    'customer_name' => 'John Doe',
    'customer_email' => 'john@example.com',
    'card_number' => '4580000000000000',
    'expiry_month' => '12',
    'expiry_year' => '25',
    'cvv' => '123',
]);

if ($result['success']) {
    // Payment successful!
    $transaction = $result['transaction'];
    echo "Payment ID: " . $transaction->id;
} else {
    // Payment failed
    echo "Error: " . $result['message'];
}
```

### Save Card for Later

```php
$result = SumitPayment::tokenizeCard([
    'card_number' => '4580000000000000',
    'expiry_month' => '12',
    'expiry_year' => '25',
    'cvv' => '123',
], auth()->id());

if ($result['success']) {
    echo "Card saved! Token ID: " . $result['token_id'];
}
```

### Pay with Saved Card

```php
$result = SumitPayment::processPaymentWithToken([
    'amount' => 100.00,
    'customer_name' => 'John Doe',
    'customer_email' => 'john@example.com',
], $tokenId);
```

### Installment Payments

```php
$result = SumitPayment::processPayment([
    'amount' => 1200.00,
    'payments_count' => 12, // 12 monthly payments
    'customer_name' => 'John Doe',
    'customer_email' => 'john@example.com',
    'card_number' => '4580000000000000',
    'expiry_month' => '12',
    'expiry_year' => '25',
    'cvv' => '123',
]);
```

## Event Handling

### Listen for Successful Payments

In `app/Providers/EventServiceProvider.php`:

```php
use Sumit\LaravelPayment\Events\PaymentCompleted;

protected $listen = [
    PaymentCompleted::class => [
        \App\Listeners\SendPaymentConfirmation::class,
    ],
];
```

Create listener:

```php
namespace App\Listeners;

use Sumit\LaravelPayment\Events\PaymentCompleted;
use Illuminate\Support\Facades\Mail;

class SendPaymentConfirmation
{
    public function handle(PaymentCompleted $event)
    {
        $transaction = $event->transaction;
        
        // Send email, update order, etc.
        Mail::to($transaction->user->email)
            ->send(new PaymentConfirmationMail($transaction));
    }
}
```

## Controller Example

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Sumit\LaravelPayment\Facades\SumitPayment;
use App\Models\Order;

class CheckoutController extends Controller
{
    public function process(Request $request)
    {
        $order = Order::create([
            'user_id' => auth()->id(),
            'total' => $request->total,
        ]);

        $result = SumitPayment::processPayment([
            'amount' => $order->total,
            'order_id' => $order->id,
            'customer_name' => auth()->user()->name,
            'customer_email' => auth()->user()->email,
            'card_number' => $request->card_number,
            'expiry_month' => $request->expiry_month,
            'expiry_year' => $request->expiry_year,
            'cvv' => $request->cvv,
        ]);

        if ($result['success']) {
            $order->update(['status' => 'paid']);
            return redirect()->route('order.success');
        }

        return back()->withErrors(['payment' => $result['message']]);
    }
}
```

## View Example

Use the provided example form or create your own:

```bash
php artisan vendor:publish --tag=sumit-payment-views
```

Then include in your blade:

```blade
@include('sumit-payment::payment-form')
```

## API Routes

The package automatically registers these routes:

- `POST /sumit/payment/process` - Process payment
- `GET /sumit/payment/callback` - Redirect callback
- `GET /sumit/tokens` - List saved cards (auth required)
- `POST /sumit/tokens` - Save new card (auth required)
- `DELETE /sumit/tokens/{id}` - Delete card (auth required)

## Testing

Enable test mode in `.env`:

```env
SUMIT_TESTING_MODE=true
```

This processes payments as authorization-only (no actual charge).

## Common Configurations

### Email Invoices

```env
SUMIT_EMAIL_DOCUMENT=true
```

### Change Language

```env
SUMIT_DOCUMENT_LANGUAGE=en  # he or en
```

### Redirect Payment Flow

```env
SUMIT_PCI_MODE=redirect
```

### Maximum Installments

```env
SUMIT_MAXIMUM_PAYMENTS=24
```

## Troubleshooting

### "Invalid credentials"

1. Check your `.env` credentials
2. Run `php artisan config:cache`
3. Verify Company ID and API Key

### "Table not found"

Run migrations:

```bash
php artisan migrate
```

### "Payment gateway is not properly configured"

Ensure all required environment variables are set:
- `SUMIT_COMPANY_ID`
- `SUMIT_API_KEY`
- `SUMIT_MERCHANT_NUMBER`

## Next Steps

📚 **Read Full Documentation:**
- [Installation Guide](INSTALLATION.md)
- [API Reference](API_REFERENCE.md)
- [Migration Guide](MIGRATION_GUIDE.md)

🔧 **Customize:**
- Extend services for custom logic
- Create event listeners
- Modify views

🧪 **Test:**
- Write unit tests
- Test with real API
- Load testing

## Support

- Email: support@sumit.co.il
- Documentation: https://help.sumit.co.il
- Issues: GitHub Issues

## Security

⚠️ **Important:**
- Always use HTTPS in production
- Never commit credentials to version control
- Regularly update the package
- Monitor transaction logs

## Complete Example

Here's a complete checkout flow:

```php
// routes/web.php
Route::post('/checkout', [CheckoutController::class, 'process'])
    ->middleware('auth');

// app/Http/Controllers/CheckoutController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Sumit\LaravelPayment\Facades\SumitPayment;

class CheckoutController extends Controller
{
    public function process(Request $request)
    {
        $validated = $request->validate([
            'amount' => 'required|numeric|min:0.01',
            'card_number' => 'required|string',
            'expiry_month' => 'required|string|size:2',
            'expiry_year' => 'required|string',
            'cvv' => 'required|string|size:3',
        ]);

        $result = SumitPayment::processPayment([
            'amount' => $validated['amount'],
            'customer_name' => auth()->user()->name,
            'customer_email' => auth()->user()->email,
            'card_number' => $validated['card_number'],
            'expiry_month' => $validated['expiry_month'],
            'expiry_year' => $validated['expiry_year'],
            'cvv' => $validated['cvv'],
            'description' => 'Purchase from ' . config('app.name'),
        ]);

        if ($result['success']) {
            session()->flash('success', 'Payment successful!');
            return redirect()->route('home');
        }

        return back()
            ->withErrors(['payment' => $result['message']])
            ->withInput();
    }
}

// app/Providers/EventServiceProvider.php
use Sumit\LaravelPayment\Events\PaymentCompleted;
use App\Listeners\SendPaymentConfirmation;

protected $listen = [
    PaymentCompleted::class => [
        SendPaymentConfirmation::class,
    ],
];

// app/Listeners/SendPaymentConfirmation.php
namespace App\Listeners;

use Sumit\LaravelPayment\Events\PaymentCompleted;
use Illuminate\Support\Facades\Log;

class SendPaymentConfirmation
{
    public function handle(PaymentCompleted $event)
    {
        Log::info('Payment completed', [
            'transaction_id' => $event->transaction->id,
            'amount' => $event->transaction->amount,
        ]);
        
        // Send email, update records, etc.
    }
}
```

That's it! You're ready to accept payments with SUMIT! 🎉

For more advanced features, check out the full documentation.
