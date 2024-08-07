# Laravel v10x, v11x Roles And Permissions Without Package 

### Step 1: Download Fresh Laravel Laravel v10x, v11x

```bash
 composer create-project laravel/laravel example-app
 ```

### Step 2:  Create Auth

```bash
     composer require laravel/ui --dev
     php artisan ui bootstrap --auth
     npm install
     npm run dev
```

### Step 3: Connect Database

```bash
     DB_CONNECTION=mysql
     DB_HOST=127.0.0.1
     DB_PORT=3306
     DB_DATABASE=laravel
     DB_USERNAME=root
     DB_PASSWORD=password
```

### Step 4: Make Model

```bash
     php artisan make:model Permission -m
     php artisan make:model Role -m
```

### Now update the migrations file like:
```bash
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('roles', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('roles');
    }
};

```

### Migrations for permissions table:

```bash
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('permissions', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('permissions');
    }
};

```


### Now time to create multiple pivot tables, We’ll create a new migration file for the table users_permissions. So run the below command to create

```bash
php artisan make:migration create_users_permissions_table
```

```bash
<?php

use App\Models\Permission;
use App\Models\User;
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('users_permissions', function (Blueprint $table) {
            $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
            $table->foreignIdFor(Permission::class)->constrained()->cascadeOnDelete();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('users_permissions');
    }
};

```

### Now let’s create a pivot table for users_roles.

```bash
 php artisan make:migration users_roles
```

```bash
<?php

use App\Models\Role;
use App\Models\User;
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('users_roles', function (Blueprint $table) {
            $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
            $table->foreignIdFor(Role::class)->constrained()->cascadeOnDelete();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('users_roles');
    }
};

```

Under a particular role, the user may have specific permission. For example, a user may have permission to post a topic, and an admin may have permission to edit or delete a topic. In this case, let’s set up a new table for roles_permissions to handle this complexity.
```bash
php artisan make:migration create_roles_permissions_table --create=roles_permissions
```

### Now update the roles_permissions like:

```bash
<?php

use App\Models\Permission;
use App\Models\Role;
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('roles_permissions', function (Blueprint $table) {
            $table->foreignIdFor(Role::class)->constrained()->cascadeOnDelete();
            $table->foreignIdFor(Permission::class)->constrained()->cascadeOnDelete();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('roles_permissions');
    }
};

```

### Now run the following command to create a migration

```bash
php artisan migrate
```

### Step 5: Update Model With Relationship
Now update the Role model with a relationship like this:
app\Models\Role.php

```bash
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Role extends Model
{
    use HasFactory;

    public function permissions()
    {
        return $this->belongsToMany(Permission::class, 'roles_permissions');
    }

    public function users()
    {
        return $this->belongsToMany(User::class, 'users_roles');
    }
}

```
### Now update the Permission model with a relationship like this:
app\Models\Permission.php
```bash
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Permission extends Model
{
    use HasFactory;

    public function roles()
    {
        return $this->belongsToMany(Role::class, 'roles_permissions');
    }

    public function users()
    {
        return $this->belongsToMany(User::class, 'users_permissions');
    }
}

```
### Step 6: Create  Service Trait
Now in this step, we will create a service or helper trait to easily handle roles and permissions in the Laravel 10 application.
App\Traits\HasPermissionsTrait.php

```bash
<?php

namespace App\Traits;

use App\Models\Role;
use App\Models\Permission;

trait HasPermissionsTrait
{
    public function givePermissionsTo(...$permissions)
    {
        $permissions = $this->getAllPermissions($permissions);
        if ($permissions === null) {
            return $this;
        }
        $this->permissions()->saveMany($permissions);
        return $this;
    }

    public function withdrawPermissionsTo(...$permissions)
    {
        $permissions = $this->getAllPermissions($permissions);
        $this->permissions()->detach($permissions);
        return $this;
    }

    public function refreshPermissions(...$permissions)
    {
        $this->permissions()->detach();
        return $this->givePermissionsTo($permissions);
    }

    public function hasPermissionTo($permission)
    {
        return $this->hasPermissionThroughRole($permission) && $this->hasPermission($permission);
    }

    public function hasPermissionThroughRole($permission)
    {
        foreach($permission->roles as $role) {
            if ($this->roles->contains($role)) {
                return true;
            }
        }
        return false;
    }

    public function hasRole(...$roles)
    {
        foreach($roles as $role) {
            if ($this->roles->contains('name', $role)) {
                return true;
            }
        }
        return false;
    }

    public function roles()
    {
        return $this->belongsToMany(Role::class, 'users_roles');
    }
    public function permissions()
    {
        return $this->belongsToMany(Permission::class, 'users_permissions');
    }
    protected function hasPermission($permission)
    {
        return (bool) $this->permissions->where('name', $permission->name)->count();
    }

    protected function getAllPermissions(array $permissions)
    {
        return Permission::whereIn('name', $permissions)->get();
    }
}
```

### Now update the User model by using this trait.
app\Models\User.php
```bash
<?php

namespace App\Models;

use App\Traits\HasPermissionsTrait;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens,
        HasFactory,
        Notifiable,
        HasPermissionsTrait;

    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
    ];
}

```


### Step 7: Create Role Blade Directive
Now this time we will create a @role() blade directive. So update the app service provider like this:
App\Providers\AppServiceProvider.php

```bash
<?php

namespace App\Providers;

use App\Models\Permission;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\Blade;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {

    }

    public function boot(): void
    {
       try {
            Permission::with('roles.users')->get()->each(function ($permission) {
                Gate::define($permission->name, function ($user) use ($permission) {
                    return $user->hasPermissionTo($permission);
                });
            });
        } catch (\Exception $e) {
            report($e);
        }

        Blade::directive('role', function ($role) {
            return "<?php if(auth()->check() && auth()->user()->hasRole({$role})) : ?>";
        });

        Blade::directive('endrole', function ($role) {
            return "<?php endif; ?>";
        });
    }
}

```

### Step 8: Create Seeder
Now in this step, we will create some dummy data to create roles and permissions with users. So run the below command:
```bash
php artisan make:seeder RoleSeeder

```

### Now update it like this:
Database\Seeders\RoleSeeder.php


```bash
<?php

namespace Database\Seeders;

use App\Models\Role;
use App\Models\User;
use App\Models\Permission;
use Illuminate\Database\Seeder;
use Illuminate\Database\Console\Seeds\WithoutModelEvents;

class RoleSeeder extends Seeder
{
    public function run(): void
    {
        $permission = new Permission();
        $permission->name = 'create-post';
        $permission->save();

        $role = new Role();
        $role->name = 'admin';
        $role->save();
        $role->permissions()->attach($permission);
        $permission->roles()->attach($role);

        $permission = new Permission();
        $permission->name = 'create-user';
        $permission->save();

        $role = new Role();
        $role->name = 'user';
        $role->save();
        $role->permissions()->attach($permission);
        $permission->roles()->attach($role);

        $admin = Role::where('name', 'admin')->first();
        $userRole = Role::where('name', 'user')->first();
        $create_post = Permission::where('name', 'create-post')->first();
        $create_user = Permission::where('name', 'create-user')->first();

        $admin = new User();
        $admin->name = 'Admin';
        $admin->email = 'admin@gmail.com';
        $admin->password = bcrypt('admin');
        $admin->save();
        $admin->roles()->attach($admin);
        $admin->permissions()->attach($create_post);

        $user = new User();
        $user->name = 'User';
        $user->email = 'user@gmail.com';
        $user->password = bcrypt('user');
        $user->save();
        $user->roles()->attach($userRole);
        $user->permissions()->attach($create_user);
    }
}

```

### Now update the database seeder like this:
DatabaseSeeder\DatabaseSeeder.php


```bash
<?php

namespace Database\Seeders;

// use Illuminate\Database\Console\Seeds\WithoutModelEvents;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call(RoleSeeder::class);
    }
}

```

### Now run the below command to generate some fake data to test our laravel 10 roles and permissions without package application.

```bash
php artisan db:seed

```

### Step 9: Create Middleware
In this step, we will create a role middleware so that we can handle user requests using middleware.
```bash
php artisan make:middleware RolePermissionMiddleware
```

### And update it like this:
App\Http\Middleware\RolePermissionMiddleware.php
```bash
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class RolePermissionMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next, $role, $permission): Response
    {
        if (!$request->user()->hasRole($role)) {
            abort(404);
        }
        if ($permission !== null && !$request->user()->can($permission)) {
            abort(404);
        }

        return $next($request);
    }
}


```


### Now register it inside the kernel.php like this:
bootstrap\app.php
App\Http\Kernel.php 

### v11x
```bash
 ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
           'role'=>  \App\Http\Middleware\RolePermissionMiddleware::class
        ]);
    })

```

### v10x 
```bash
 protected $middlewareAliases = [
        'role' =>  \App\Http\Middleware\RolePermissionMiddleware::class,
 ]

 ```

### Now all is set. We can check user requests and roles like this:
     resources/views/home.blade.php
```bash
@extends('layouts.app')

@section('content')
<div class="container">
    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card">
                <div class="card-header">{{ __('Dashboard') }}</div>

                <div class="card-body">
                    @if (session('status'))
                        <div class="alert alert-success" role="alert">
                            {{ session('status') }}
                        </div>
                    @endif
                    @role('admin')
                        {{ __('You are admin') }}
                    @endrole
                    @role('user')
                        {{ __('You are user') }}
                    @endrole
                </div>
            </div>
        </div>
    </div>
</div>
@endsection

```

### Check user roles and permissions in the controller:


```bash
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class TutorialController extends Controller
{
    public function __invoke()
    {
        if (auth()->user()->can('create-user')) {
            return view('welcome');
        }
    }
}

```

### Use in your route like this:
routes/web.php

```bash
<?php

use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\TutorialController;

/*
|--------------------------------------------------------------------------
| Web Routes
|--------------------------------------------------------------------------
|
| Here is where you can register web routes for your application. These
| routes are loaded by the RouteServiceProvider and all of them will
| be assigned to the "web" middleware group. Make something great!
|
*/

Route::get('/', TutorialController::class)->middleware('role:admin');

```

### Use in a group route:


```bash
Route::group(['middleware' => 'role:admin'], function() {
   //
});

Route::group(['middleware' => 'role:user'], function() {
   //
});
```
