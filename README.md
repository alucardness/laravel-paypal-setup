<h1>Laravel Paypal Setup with Omni Library</h1>

<p>This README.md file provides instructions on how to set up Laravel with the OmniPay library to enable payments through Paypal.</p>

<h2>Prerequisites</h2>

<p>Before starting with the setup, make sure you have the following prerequisites installed:</p>

<ul>
    <li><a href="https://laravel.com/docs/10.x">Laravel</a></li>
    <li><a href="https://getcomposer.org/doc/00-intro.md">Composer</a></li>
</ul>

<h2>Installation</h2>

1.Install the OmniPay library using composer by running the following command in your Laravel project's root directory:
<br>

```javascript
composer require league/omnipay
```

2.Install the PayPal driver for OmniPay using composer by running the following command:
<br>

```javascript
composer require league/omnipay-paypal
```

3.Once the installation is complete, open your Laravel project's .env file and add your PayPal API credentials. The credentials are required to connect to the PayPal API and process payments. The credentials can be obtained from your PayPal <a href="https://developer.paypal.com/">Paypal Developer</a> account.

<ol>
    <li>Navigate to the <code>Apps & Credentials</code> tab.</li>
    <li>Create App.</li>
    <li>Set a name.</li>
    <li>Copy the <code>Client ID</code> and <code>Secret</code>.</li>
</ol>

<br>

```makefile
PAYPAL_CLIENT_ID=
PAYPAL_CLIENT_SECRET=
PAYPAL_CURRENCY=USD
```

4.Create the routes
<br>

```php
Route::get('payment', [PaymentController::class, 'index']);
Route::post('charge', [PaymentController::class, 'charge']);
Route::get('success', [PaymentController::class, 'success']);
Route::get('error', [PaymentController::class, 'error']);
```

5.Create the migration and the model
<br>

Migration:

```php
    public function up(): void
    {
        Schema::create('payments', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('payment_id');
            $table->string('payer_id');
            $table->string('payer_email');
            $table->float('amount', 10, 2);
            $table->string('currency');
            $table->string('payment_status');
            $table->timestamps();
        });
    }
```

Model:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Payment extends Model
{
}

```

6.Create the methods of your PaymentController:
<br>

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Omnipay\Omnipay;
use App\Models\Payment;

class PaymentController extends Controller
{
    private $gateway;

    public function __construct()
    {
        $this->gateway = Omnipay::create('PayPal_Rest');
        $this->gateway->setClientId(env('PAYPAL_CLIENT_ID'));
        $this->gateway->setSecret(env('PAYPAL_CLIENT_SECRET'));
        $this->gateway->setTestMode(true);  //set it to 'false' when go live
    }

    /**
     * Call a view.
     */
    public function index()
    {
        return view('payment');
    }

    /**
     * Initiate a payment on PayPal.
     *
     * @param  \Illuminate\Http\Request  $request
     */
    public function charge(Request $request)
    {
        if ($request->input('submit')) {
            try {
                $response = $this->gateway->purchase(array(
                    'amount' => $request->input('amount'),
                    'items'  => [
                        [
                            'name'        => 'Champion Ship Course',
                            'price'       => $request->input('amount'),
                            'description' => 'Get access to premium courses.',
                            'quantity'    => 1
                        ],
                    ],
                    'currency'  => env('PAYPAL_CURRENCY'),
                    'returnUrl' => url('success'),
                    'cancelUrl' => url('error'),
                ))->send();

                if ($response->isRedirect()) {
                    $response->redirect();  // this will automatically forward the customer
                } else {
                    // not successful
                    return $response->getMessage();
                }
            } catch (Exception $e) {
                return $e->getMessage();
            }
        }
    }

    /**
     * Charge a payment and store the transaction.
     *
     * @param  \Illuminate\Http\Request  $request
     */
    public function success(Request $request)
    {
        // Once the transaction has been approved, we need to complete it.
        if ($request->input('paymentId') && $request->input('PayerID')) {
            $transaction = $this->gateway->completePurchase(array(
                'payer_id'             => $request->input('PayerID'),
                'transactionReference' => $request->input('paymentId'),
            ));
            $response = $transaction->send();

            if ($response->isSuccessful()) {
                // The customer has successfully paid.
                $arr_body = $response->getData();

                // Insert transaction data into the database
                $payment                 = new Payment;
                $payment->payment_id     = $arr_body['id'];
                $payment->payer_id       = $arr_body['payer']['payer_info']['payer_id'];
                $payment->payer_email    = $arr_body['payer']['payer_info']['email'];
                $payment->amount         = $arr_body['transactions'][0]['amount']['total'];
                $payment->currency       = env('PAYPAL_CURRENCY');
                $payment->payment_status = $arr_body['state'];
                $payment->save();

                return "Payment is successful. Your transaction id is: " . $arr_body['id'];
            } else {
                return $response->getMessage();
            }
        } else {
            return 'Transaction is declined';
        }
    }

    /**
     * Error Handling.
     */
    public function error()
    {
        return 'User cancelled the payment.';
    }
}
```

7.Create the testing blade view:
<br>

```php
<form action="{{ url('charge') }}" method="post">
    <input type="text" name="amount" />
    {{ csrf_field() }}
    <input type="submit" name="submit" value="Pay Now">
</form>
```

7.For <code>Production</code> mode:
<br>

<p>Set the test mode to false in the <code>constructor</code>.</p>

```php
$this->gateway->setTestMode(false);
```

<p>and change your <code>credentials</code> in the <code>.env</code> file with your production ones.</p>
