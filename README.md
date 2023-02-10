# laravel-socialstream-encrypted

This is an example [Laravel](https://laravel.com) application that uses [Socialstream](https://github.com/joelbutcher/socialstream) for social authentication and [Crudly/Encrypted](https://github.com/Crudly/Encrypted) to encrypt social secrets in the database.

It is good practice to encrypt sensitive values in your database. If an attacker would gain access to your database (and not your application server that holds your encryption key), that would only have access to useless encrypted values from your database.

**Note that this doesn't mean passwords!** Passwords should be hashed and never encrypted. Let Laravel handle that. This is only meant for values that need to be used decrypted, like in this case social authentication tokens.

## Installation

You can clone this repo, but I don't expect to routinely keep it up-to-date, so you should instead probably recreate this from scratch in order to get the most recent packages.

-   Create a new Laravel application.

```bash
curl -s "https://laravel.build/laravel-jetsream-encrypted" | bash
```

-   Start the application.

```bash
./vendor/bin/sail up
```

-   Install the Socialstream and Encrypted packages.

```bash
./vendor/bin/sail composer require joelbutcher/socialstream crudly/encrypted
```

-   Install Socialstream [per the documentation](https://docs.socialstream.dev/getting-started/installation). Here, I've chosen the Inertia & SSR version.

```bash
./vendor/bin/sail artisan socialstream:install --stack=inertia --ssr
```

-   DO NOT YET RUN MIGRATIONS.

-   Modify `app/models/ConnectedAccounts.php` to use `Encrypted` to encrypt the sensitive fields.

```diff
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasTimestamps;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use JoelButcher\Socialstream\ConnectedAccount as SocialstreamConnectedAccount;
use JoelButcher\Socialstream\Events\ConnectedAccountCreated;
use JoelButcher\Socialstream\Events\ConnectedAccountDeleted;
use JoelButcher\Socialstream\Events\ConnectedAccountUpdated;
+ use Crudly\Encrypted\Encrypted;

class ConnectedAccount extends SocialstreamConnectedAccount
{
    use HasFactory;
    use HasTimestamps;

    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = [
        'provider',
        'provider_id',
        'name',
        'nickname',
        'email',
        'avatar_path',
        'token',
        'refresh_token',
        'expires_at',
    ];

    /**
     * The event map for the model.
     *
     * @var array
     */
    protected $dispatchesEvents = [
        'created' => ConnectedAccountCreated::class,
        'updated' => ConnectedAccountUpdated::class,
        'deleted' => ConnectedAccountDeleted::class,
    ];

+    /**
+     * The casts for the model.
+     *
+     * @var array
+     */
+    protected $casts = [
+        'token' => Encrypted::class,
+        'secret' => Encrypted::class,
+        'refresh_token' => Encrypted::class,
+    ];
}
```

-   Modify the `app/database/migrations/2020_12_22_000000_create_connected_accounts_table.php` to ensure that the sensitive columns are large enough to accept encrypted values. This is likely not strictly necessary depending upon which providers you use for social authentication, but I'd prefer to do this rather than mistakenly overflow the column.

```diff
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateConnectedAccountsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('connected_accounts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id');
            $table->string('provider');
            $table->string('provider_id');
            $table->string('name')->nullable();
            $table->string('nickname')->nullable();
            $table->string('email')->nullable();
            $table->string('telephone')->nullable();
            $table->text('avatar_path')->nullable();
-             $table->string('token', 1000);
+             $table->text('token');
-             $table->string('secret')->nullable(); // OAuth1
+             $table->text('secret')->nullable(); // OAuth1
-             $table->string('refresh_token', 1000)->nullable(); // OAuth2
+             $table->text('refresh_token')->nullable(); // OAuth2
            $table->dateTime('expires_at')->nullable(); // OAuth2
            $table->timestamps();

            $table->index(['user_id', 'id']);
            $table->index(['provider', 'provider_id']);
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('connected_accounts');
    }
}
```

-   Run migrations.

```bash
./vendor/bin/sail artisan migrate
```

-   Continue to setup your provider as per [the Socialstream documentation](https://docs.socialstream.dev/getting-started/configuration).

-   Register with your provider, query the `connected_accounts` table, and you'll see that the `token`, `secret`, and `refresh_token` columns are automatically encrypted in your database. They will also be automatically decrypted by the `Encrypted` package when they need to be used by `Socialstream`.
