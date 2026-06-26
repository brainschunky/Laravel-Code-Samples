# Laravel Code Samples

Production-grade patterns I use across projects. These aren't textbook examples — they're shaped by real bugs, scale issues, and a few 3am incidents.

---

## Table of Contents

1. [Authentication & Authorization](#1-authentication--authorization)
2. [Service Layer](#2-service-layer)
3. [Repository Pattern](#3-repository-pattern)
4. [REST API](#4-rest-api)
5. [API Resources & Transformers](#5-api-resources--transformers)
6. [Form Request Validation](#6-form-request-validation)
7. [Queue Jobs](#7-queue-jobs)
8. [Event-Driven Architecture](#8-event-driven-architecture)
9. [Payment Gateway Integration](#9-payment-gateway-integration)
10. [Dynamic Query Filtering](#10-dynamic-query-filtering)
11. [File Upload Service](#11-file-upload-service)
12. [Database Transactions](#12-database-transactions)

---

## 1. Authentication & Authorization

### JWT Authentication with Refresh Token Rotation

Most projects I work on need stateless auth. I pair `tymon/jwt-auth` with a custom `AuthService` to keep controllers thin and token logic centralized.

```php
namespace App\Services;

use App\Models\User;
use App\Models\RefreshToken;
use App\Exceptions\InvalidCredentialsException;
use Illuminate\Support\Facades\Hash;
use Tymon\JWTAuth\Facades\JWTAuth;

class AuthService
{
    public function login(array $credentials): array
    {
        $user = User::where('email', $credentials['email'])->first();

        if (! $user || ! Hash::check($credentials['password'], $user->password)) {
            throw new InvalidCredentialsException();
        }

        $access  = JWTAuth::fromUser($user);
        $refresh = $this->issueRefreshToken($user);

        return compact('access', 'refresh', 'user');
    }

    public function refresh(string $refreshToken): array
    {
        $user = $this->validateRefreshToken($refreshToken);

        // Rotate: old token is invalidated on use
        $this->revokeRefreshToken($refreshToken);

        $access  = JWTAuth::fromUser($user);
        $refresh = $this->issueRefreshToken($user);

        return compact('access', 'refresh');
    }

    public function logout(): void
    {
        JWTAuth::invalidate(JWTAuth::getToken());
    }

    private function issueRefreshToken(User $user): string
    {
        $token = bin2hex(random_bytes(40));

        $user->refreshTokens()->create([
            'token'      => hash('sha256', $token),
            'expires_at' => now()->addDays(30),
        ]);

        return $token;
    }

    private function validateRefreshToken(string $token): User
    {
        $hashed = hash('sha256', $token);
        $record = RefreshToken::where('token', $hashed)->where('expires_at', '>', now())->whereNull('revoked_at')->firstOrFail();
        return $record->user;
    }

    private function revokeRefreshToken(string $token): void
    {
        RefreshToken::where('token', hash('sha256', $token))->update(['revoked_at' => now()]);
    }
}
```

```php
// AuthController 
class AuthController extends Controller
{
    public function __construct(private AuthService $auth) {}

    public function login(LoginRequest $request): JsonResponse
    {
        $result = $this->auth->login($request->validated());
        return response()->json($result);
    }

    public function refresh(Request $request): JsonResponse
    {
        $result = $this->auth->refresh($request->bearerToken());
        return response()->json($result);
    }

    public function logout(): JsonResponse
    {
        $this->auth->logout();
        return response()->json(['message' => 'Logged out']);
    }
}
```

---

### Sanctum for SPA + Mobile (same API, two guard strategies)

```php
// config/auth.php — two separate guards
'guards' => [
    'web'    => ['driver' => 'session',  'provider' => 'users'],
    'sanctum' => ['driver' => 'sanctum', 'provider' => 'users'],
],
```

```php
// Issuing tokens with abilities (scopes)
public function tokenForMobile(User $user): string
{
    return $user->createToken(
        name: 'mobile-app',
        abilities: ['orders:read', 'orders:write', 'profile:read']
    )->plainTextToken;
}
```

```php
// Middleware checks
Route::middleware(['auth:sanctum', 'ability:orders:write'])->group(function () {
    Route::post('/orders', [OrderController::class, 'store']);
});
```

---

### Role & Permission Gates (Policy-first approach)

I avoid `if ($user->role === 'admin')` scattered across controllers. Everything goes through Policies.

```php
namespace App\Policies;
use App\Models\{User, Order};

class OrderPolicy
{
    public function view(User $user, Order $order): bool
    {
        return $user->id === $order->user_id || $user->hasRole('admin');
    }

    public function refund(User $user, Order $order): bool
    {
        return $user->hasRole(['admin', 'finance']) && $order->isEligibleForRefund();
    }
}
```

```php
// Controller — one line auth check
public function refund(Order $order): JsonResponse
{
    $this->authorize('refund', $order);
}
```

---

## 2. Service Layer

Business logic lives in services, not controllers or models. Controllers handle HTTP. Models handle relationships. Services handle everything in between.

```php
namespace App\Services;
use App\Models\Subscription;
use App\Events\SubscriptionCreated;
use App\Exceptions\PlanNotFoundException;
use Illuminate\Support\Facades\DB;

class SubscriptionService
{
    public function create(array $data): Subscription
    {
        return DB::transaction(function () use ($data) {
            $plan = Plan::findOrFail($data['plan_id']);

            if (! $plan->isAvailableForUser($data['user_id'])) {
                throw new PlanNotFoundException("Plan unavailable for this user.");
            }

            $subscription = Subscription::create([
                'user_id'   => $data['user_id'],
                'plan_id'   => $plan->id,
                'starts_at' => now(),
                'ends_at'   => now()->addDays($plan->duration_days),
                'status'    => 'active',
            ]);
            event(new SubscriptionCreated($subscription));
            return $subscription;
        });
    }

    public function cancel(Subscription $subscription, string $reason = null): void
    {
        DB::transaction(function () use ($subscription, $reason) {
            $subscription->update([
                'status'       => 'cancelled',
                'cancelled_at' => now(),
                'cancel_reason' => $reason,
            ]);
            event(new SubscriptionCancelled($subscription));
        });
    }
}
```

---

## 3. Repository Pattern

I use repositories when a project has complex queries that would otherwise get duplicated across multiple services. For simpler projects, scopes on the model are enough.

```php
namespace App\Repositories;
use App\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

class UserRepository
{
    public function activePaginated(int $perPage = 20): LengthAwarePaginator
    {
        return User::query()->where('status', 'active')->with(['roles', 'profile'])->latest()->paginate($perPage);
    }

    public function findByEmail(string $email): ?User
    {
        return User::where('email', $email)->first();
    }

    public function create(array $data): User
    {
        return User::create($data);
    }

    public function updateLastLogin(User $user): void
    {
        $user->update(['last_login_at' => now()]);
    }
}
```

---

## 4. REST API

```php
public function index(Request $request): AnonymousResourceCollection
{
    $users = User::query()
        ->when($request->filled('search'), function ($q) use ($request) {
            $q->where(function ($inner) use ($request) {
                $inner->where('name',  'like', "%{$request->search}%")
                ->orWhere('email', 'like', "%{$request->search}%");
            });
        })
        ->when($request->filled('role'), function ($q) use ($request) {
            $q->whereHas('roles', fn ($r) => $r->where('name', $request->role));
        })->with(['roles', 'profile'])->latest()->paginate(15);
    return UserResource::collection($users);
}

public function show(User $user): UserResource
{
    return new UserResource($user->load(['roles', 'orders', 'profile']));
}

public function store(StoreUserRequest $request): JsonResponse
{
    $user = $this->userService->create($request->validated());
    return (new UserResource($user))->response()->setStatusCode(201);
}

public function destroy(User $user): JsonResponse
{
    $this->authorize('delete', $user);
    $this->userService->delete($user);
    return response()->json(null, 204);
}
```

---

## 5. API Resources & Transformers

```php
namespace App\Http\Resources;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'email'      => $this->email,
            'avatar'     => $this->profile?->avatar_url,
            'roles'      => RoleResource::collection($this->whenLoaded('roles')),
            'created_at' => $this->created_at->toISOString(),

            // Only include in admin context
            'metadata'   => $this->when(
                $request->user()?->hasRole('admin'),
                fn () => [
                    'last_login_at' => $this->last_login_at?->toISOString(),
                    'ip_address'    => $this->last_ip,
                ]
            ),
        ];
    }
}
```

---

## 6. Form Request Validation

```php
namespace App\Http\Requests;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rules\Password;

class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->hasRole('admin');
    }

    public function rules(): array
    {
        return [
            'name'     => ['required', 'string', 'max:100'],
            'email'    => ['required', 'email', 'unique:users,email'],
            'password' => ['required', Password::min(8)->mixedCase()->numbers()->symbols()],
            'role'     => ['required', 'exists:roles,name'],
        ];
    }

    public function messages(): array
    {
        return [
            'email.unique' => 'This email is already registered.',
            'role.exists'  => 'Selected role does not exist.',
        ];
    }

    protected function prepareForValidation(): void
    {
        $this->merge([
            'email' => strtolower(trim($this->email)),
        ]);
    }
}
```

---

## 7. Queue Jobs

```php
namespace App\Jobs;
use App\Mail\WelcomeMail;
use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\{InteractsWithQueue, SerializesModels};
use Illuminate\Support\Facades\Mail;

class SendWelcomeMail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries   = 3;
    public int $backoff = 60; 

    public function __construct(public readonly User $user) {}

    public function handle(): void
    {
        Mail::to($this->user->email)->send(new WelcomeMail($this->user));
    }

    public function failed(\Throwable $e): void
    {
        \Log::error("WelcomeMail failed for user {$this->user->id}: {$e->getMessage()}");
    }
}
```

Dispatching with a delay:

```php
SendWelcomeMail::dispatch($user)->delay(now()->addSeconds(10));
```

---

## 8. Event-Driven Architecture

```php

class OrderShipped
{
    public function __construct(
        public readonly Order $order,
        public readonly string $trackingNumber
    ) {}
}

// Listener 1 — notify customer
class NotifyCustomerOnShipment
{
    public function handle(OrderShipped $event): void
    {
        $event->order->user->notify(
            new ShipmentNotification($event->order, $event->trackingNumber)
        );
    }
}

// Listener 2 — update warehouse stock
class UpdateInventoryOnShipment
{
    public function handle(OrderShipped $event): void
    {
        foreach ($event->order->items as $item) {
            $item->product->decrement('stock', $item->quantity);
        }
    }
}

protected $listen = [
    OrderShipped::class => [
        NotifyCustomerOnShipment::class,
        UpdateInventoryOnShipment::class,
    ],
];
```

---

## 9. Payment Gateway Integration

I wrap payment SDKs in a dedicated service and interface so swapping providers doesn't touch business logic.

```php
interface PaymentGatewayInterface
{
    public function charge(array $payload): PaymentResult;
    public function refund(string $transactionId, int $amount): bool;
}

class StripePaymentGateway implements PaymentGatewayInterface
{
    public function charge(array $payload): PaymentResult
    {
        try {
            $intent = \Stripe\PaymentIntent::create([
                'amount'   => $payload['amount'],
                'currency' => $payload['currency'] ?? 'inr',
                'metadata' => ['order_id' => $payload['order_id']],
            ]);

            return new PaymentResult(
                success: true,
                transactionId: $intent->id,
                status: $intent->status
            );
        } catch (\Stripe\Exception\CardException $e) {
            return new PaymentResult(success: false, error: $e->getMessage());
        }
    }

    public function refund(string $transactionId, int $amount): bool
    {
        \Stripe\Refund::create([
            'payment_intent' => $transactionId,
            'amount'         => $amount,
        ]);

        return true;
    }
}
```

```php

class OrderPaymentService
{
    public function __construct(private PaymentGatewayInterface $gateway) {}

    public function processPayment(Order $order, array $paymentData): void
    {
        DB::transaction(function () use ($order, $paymentData) {
            $result = $this->gateway->charge([
                'amount'   => $order->total_amount,
                'currency' => 'inr',
                'order_id' => $order->id,
            ]);

            if (! $result->success) {
                throw new PaymentFailedException($result->error);
            }

            $order->update(['payment_status' => 'paid']);

            Payment::create([
                'order_id'       => $order->id,
                'transaction_id' => $result->transactionId,
                'amount'         => $order->total_amount,
                'gateway'        => 'stripe',
            ]);

            GenerateInvoice::dispatch($order);
        });
    }
}
```

---

## 10. Dynamic Query Filtering

For APIs with many optional filters, I use a dedicated `Filter` class per model instead of stacking `when()` calls in the controller.

```php

namespace App\Filters;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Http\Request;

class UserFilter
{
    public function __construct(private Request $request) {}

    public function apply(Builder $query): Builder
    {
        return $query
            ->when($this->request->filled('status'), fn ($q) =>
                $q->where('status', $this->request->status)
            )
            ->when($this->request->filled('role'), fn ($q) =>
                $q->whereHas('roles', fn ($r) =>
                    $r->where('name', $this->request->role)
                )
            )
            ->when($this->request->filled('from'), fn ($q) =>
                $q->whereDate('created_at', '>=', $this->request->from)
            )
            ->when($this->request->filled('to'), fn ($q) =>
                $q->whereDate('created_at', '<=', $this->request->to)
            );
    }
}

$users = (new UserFilter($request))->apply(User::query())->paginate(20);

```

---

## 11. File Upload Service

```php

namespace App\Services;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class FileUploadService
{
    public function upload(UploadedFile $file, string $folder, string $disk = 'public'): string
    {
        $filename = Str::uuid() . '.' . $file->getClientOriginalExtension();
        return $file->storeAs($folder, $filename, $disk);
    }

    public function uploadImage(UploadedFile $file, string $folder): string
    {
        if (! in_array($file->getMimeType(), ['image/jpeg', 'image/png', 'image/webp'])) {
            throw new \InvalidArgumentException('Unsupported image type.');
        }
        return $this->upload($file, $folder);
    }

    public function delete(string $path, string $disk = 'public'): bool
    {
        return Storage::disk($disk)->delete($path);
    }
}
```

---

## 12. Database Transactions

```php

DB::transaction(function () use ($order, $payment) {
    $order->update(['payment_status' => 'paid']);

    Payment::create([
        'order_id'       => $order->id,
        'transaction_id' => $payment['transaction_id'],
        'amount'         => $payment['amount'],
    ]);
    
    GenerateInvoice::dispatchAfterResponse($order);
});
```

For operations across multiple services where you want manual control:

```php
$db = DB::connection();
$db->beginTransaction();

try {
    $this->inventoryService->reserve($items);
    $this->orderService->create($orderData);
    $this->notificationService->schedule($user);

    $db->commit();
} catch (\Throwable $e) {
    $db->rollBack();
    \Log::error('Order creation failed', ['error' => $e->getMessage()]);
    throw $e;
}
```

---

*These samples cover patterns I reach for on most mid-to-large Laravel projects. The authentication section in particular tends to vary most by project requirements — happy to discuss trade-offs for your specific case.*
