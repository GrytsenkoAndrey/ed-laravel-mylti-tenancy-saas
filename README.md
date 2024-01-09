# ed-laravel-mylti-tenancy-saas
Laravel — Multi Tenancy with Single Database (SaaS)

One of the fundamental decisions to make when developing SaaS projects is whether to manage tenants through a single database or multiple databases. This article aims to strengthen the perspective of those who decide in favor of a “Single Database” approach and demonstrate how to proceed on this path more easily.

In the Single Database scenario, there is a parameter that distinguishes tenants from each other for almost every table. This parameter can be named ‘tenant_id,’ ‘customer_id,’ ‘corporate_id,’ ‘user_id,’ and so on.

Continuing with this scenario, the next step is to add this isolator to all your WHERE clauses or create methods. While validating and adding this individually to queries is one method, it is admittedly not an optimal one.

So, how can we simplify this process?

By adding Traits and Scopes to our models. In other words, you can use Traits that automatically fill in these fields, whether you add ‘tenant_id’ as a parameter to your requests or not. This approach will enhance system security without burdening you with code clutter.

You can implement this as follows:

Let’s say we have a User entity in our scenario, and this entity has a parameter named “tenant_id.” We use this ‘tenant_id’ to determine what to show or not show to an authorized user.

First, let’s create our Trait.

```
<?php
namespace App\Traits;

trait TenantAttributeTrait
{
    public static function bootTenantAttributeTrait()
    {
        static::creating(function ($model) {
            $model->tenant_id = auth()->user()->tenant_id;
        });
        static::retrieved(function ($model) {
            $model->where('tenant_id', auth()->user()->tenant_id);
        });
    }
}
```

The above trait will automatically add the ‘tenant_id’ during our “create” method. The crucial point here is that the trait we applied overrides everything. Even if a user manages to send data like ‘tenant_id’ => 15, we will perform the content entry by taking the user’s tenant into account.

The “retrieved” method is only valid for values like find and first(), ensuring the security of our operation.

As you may have noticed, there are no scenarios here for all() and get(). We will handle them in a separate trait. First, let’s add them to our model.

```
class Company extends AbstractModel
{
    use TenantAttributeTrait;
```

After implementing this, from now on, all new data entries into the “Company” model will automatically have the tenant ID of the authorized user. Similarly, in the use of find and first(), the WHERE clause will automatically include the ‘tenant_id’ variable.

We have elevated both our security and code cleanliness by a notch.

However, it’s not over yet.

Because we also need to apply this to get and all methods.

First, let’s open a scope.

```
<?php
namespace App\Scopes;

use Illuminate\Database\Eloquent\Scope;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model)
    {
        if(auth()->user()){
            $builder->where('tenant_id', auth()->user()->tenant_id);
        }
    }
}
```

Next, we will use this scope in our new trait.

```
<?php
namespace App\Traits;

use App\Scopes\TenantScope;
trait TenantScoped
{
    public static function bootTenantScoped()
    {
        static::addGlobalScope(new TenantScope);
    }
}
```

And then, we add this trait to our model from a while ago.

```
class Company extends AbstractModel
{
    use TenantAttributeTrait, TenantScoped;
```

And the process is complete. Now, your queries like Company::all() work as follows: ```select * from companies where tenant_id='blabla'.```

You’ve enhanced the security of your multi-tenant system a bit more and freed yourself from code clutter.

What about test cases?

That depends on your structure because these traits are automatically added whenever the model is triggered. If you have test scenarios where you don’t want ‘tenant_id’ to be triggered automatically, you can write exceptions that exclude traits in those scenarios. Or, in scenarios where you read data from 3–4 different tenants with a single user, you can enhance your traits.

