# Laravel Code Samples

This repository contains production-inspired Laravel code samples demonstrating clean architecture, enterprise design patterns, RESTful APIs, payment integrations, queues, events, and scalable backend development practices.

The examples are written from scratch for learning and portfolio purposes and do not contain proprietary client code.

---

# Table of Contents

1. Service Layer Architecture
2. Repository Pattern
3. REST API Development
4. API Resources
5. Form Request Validation
6. Queue Jobs
7. Event Driven Architecture
8. Payment Gateway Integration
9. Dynamic Query Filtering
10. File Upload Service
11. Database Transactions
12. Custom Helper Functions

---

# 1. Service Layer Architecture

### Description

Business logic should remain outside controllers to keep applications clean, maintainable, and testable. The Service Layer centralizes domain logic and allows controllers to focus only on handling HTTP requests.

```php
namespace App\Services;

use App\Models\Subscription;
use Illuminate\Support\Facades\DB;

class SubscriptionService
{
    public function create(array $data)
    {
        return DB::transaction(function () use ($data) {

            $subscription = Subscription::create([
                'user_id'   => $data['user_id'],
                'plan_id'   => $data['plan_id'],
                'starts_at' => now(),
                'ends_at'   => now()->addMonth()
            ]);

            event(new SubscriptionCreated($subscription));

            return $subscription;
        });
    }
}
```

### Highlights

- Service Layer Architecture
- Database Transactions
- Event Driven Development
- Single Responsibility Principle
- Maintainable Business Logic

---

# 2. Repository Pattern

### Description

Repositories abstract the data layer, making business logic independent from Eloquent and improving maintainability.

```php
class UserRepository
{
    public function getActiveUsers()
    {
        return User::where('status',1)
                    ->latest()
                    ->paginate(20);
    }

    public function find($id)
    {
        return User::findOrFail($id);
    }

    public function create(array $data)
    {
        return User::create($data);
    }
}
```

### Highlights

- Repository Pattern
- Reusable Data Layer
- Separation of Concerns
- Easy Unit Testing

---

# 3. REST API Development

### Description

A clean REST API with eager loading, searching, pagination, and API Resource transformation.

```php
public function index(Request $request)
{
    $users = User::query();

    if ($request->filled('search')) {
        $users->where(function ($query) use ($request) {

            $query->where('name','like','%'.$request->search.'%')
                  ->orWhere('email','like','%'.$request->search.'%');

        });
    }

    return UserResource::collection(
        $users->with('roles')
              ->latest()
              ->paginate(15)
    );
}
```

### Highlights

- RESTful API Design
- Search Filtering
- Pagination
- Eloquent Relationships
- API Resources

---

# 4. Database Transactions

### Description

Critical business operations should execute atomically to maintain data consistency.

```php
DB::transaction(function () use ($order, $payment) {

    $order->update([
        'payment_status' => 'paid'
    ]);

    Payment::create([
        'order_id' => $order->id,
        'amount' => $payment['amount']
    ]);

    Invoice::dispatch($order);

});
```

### Highlights

- Transaction Management
- Data Integrity
- Queue Integration
- Reliable Processing

---

# 5. Queue Jobs

### Description

Heavy operations such as emails and notifications should be processed asynchronously.

```php
class SendWelcomeMail implements ShouldQueue
{
    use Dispatchable, Queueable;

    public function __construct(
        public User $user
    ){}

    public function handle()
    {
        Mail::to($this->user->email)
            ->send(new WelcomeMail($this->user));
    }
}
```

### Highlights

- Queue Processing
- Background Jobs
- Improved Performance
- Scalable Applications

---

# 6. Dynamic Query Filtering

### Description

Flexible query building allows APIs to support filtering without duplicated conditions.

```php
$query = User::query();

$query->when(request('status'), function ($q) {
    $q->where('status', request('status'));
});

$query->when(request('role'), function ($q) {
    $q->whereHas('roles', function ($role) {
        $role->where('name', request('role'));
    });
});

return $query->paginate(20);
```

### Highlights

- Clean Query Builder
- Dynamic Filters
- Reusable Queries
- Readable Code

---

# 7. File Upload Service

### Description

A reusable service class for uploading files with unique naming.

```php
class FileUploadService
{
    public function upload($file, $folder)
    {
        $name = time().'_'.$file->getClientOriginalName();

        return $file->storeAs($folder, $name, 'public');
    }
}
```

### Highlights

- Service Class
- Reusable Logic
- Clean Architecture
- File Management

---

# Best Practices Followed

- Clean Architecture
- SOLID Principles
- Repository Pattern
- Service Layer
- RESTful API Design
- Dependency Injection
- Event Driven Development
- Database Transactions
- Queue Processing
- Reusable Components
- Secure Coding Practices
- Laravel Coding Standards
