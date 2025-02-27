# Filament OTP Login Setup Guide

This guide walks you through setting up one-time password (OTP) authentication in your Filament admin panel using the `afsakar/filament-otp-login` package.

## 📋 Overview

This implementation provides:

- OTP (One-Time Password) authentication for Filament admin panel
- Customizable OTP expiration time
- Ability to exempt specific users from OTP requirements
- Session-based OTP to reduce repetitive authentication

## 🚀 Installation

### 1. Create a new Laravel project

```bash
composer create-project laravel/laravel filament-otp --prefer-dist
```

### 2. Configure Email

Set up Mailtrap or your preferred email provider in your `.env` file:

```
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=your_username
MAIL_PASSWORD=your_password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

### 3. Install the OTP package

```bash
composer require afsakar/filament-otp-login
```

### 4. Publish and run migrations

```bash
php artisan vendor:publish --tag="filament-otp-login-migrations"
php artisan migrate
```

### 5. (Optional) Publish config and translation files

```bash
php artisan vendor:publish --tag="filament-otp-login-config"
php artisan vendor:publish --tag="filament-otp-login-translations"
```

## ⚙️ Configuration

### Register the plugin

Add the plugin to your Filament panel provider:

```php
use Afsakar\FilamentOtpLogin\FilamentOtpLoginPlugin;

public function panel(Panel $panel): Panel
{
    return $panel
        ->plugins([
            FilamentOtpLoginPlugin::make(),
        ]);
}
```

### (Optional) Add Password Reset Functionality

To enable password reset functionality in your Filament panel:

```php
public function panel(Panel $panel): Panel
{
    return $panel
        ->passwordReset()
        // other configurations
        ->plugins([
            FilamentOtpLoginPlugin::make(),
        ]);
}
```

> **Note:** Password reset requires proper email configuration and running `php artisan queue:work` to process the email sending queue.

## 🔧 Advanced Customization

### Exempt specific users from OTP

Implement the `CanLoginDirectly` trait in your User model to skip OTP for certain users:

```php
use Afsakar\FilamentOtpLogin\Models\Contracts\CanLoginDirectly;

class User extends Authenticatable implements CanLoginDirectly
{
    use HasFactory, Notifiable;

    // other user model code

    public function canLoginDirectly(): bool
    {
        return str($this->email)->endsWith('@example.com');
    }
}
```

### Time-based OTP bypass

To implement a session-based approach where users only need OTP once every certain period:

1. Create a migration to add a last login timestamp:

```bash
php artisan make:migration add_last_login_at_to_users_table --table=users
```

2. Define the migration:

```php
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->timestamp('last_login_at')->nullable();
    });
}

public function down()
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('last_login_at');
    });
}
```

3. Add the field to your User model's fillable properties:

```php
protected $fillable = [
    'name',
    'email',
    'password',
    'last_login_at'
];
```

4. Update the login behavior to track the login time by modifying the `Login.php` file:

```php
protected function doLogin(): void
{
    $data = $this->form->getState();

    if (! Filament::auth()->attempt($this->getCredentialsFromFormData($data), $data['remember'] ?? false)) {
        $this->throwFailureValidationException();
    }

    $user = Filament::auth()->user();

    if (
        ($user instanceof FilamentUser) &&
        (! $user->canAccessPanel(Filament::getCurrentPanel()))
    ) {
        Filament::auth()->logout();

        $this->throwFailureValidationException();
    }

    $user->forceFill(['last_login_at' => now()])->save();

    session()->regenerate();
}
```

5. Add the time-based logic to your User model:

```php
use Carbon\Carbon;

public function canLoginDirectly(): bool
{
    return $this->last_login_at && Carbon::parse($this->last_login_at)->diffInMinutes(now()) <= 20; // Skip OTP if logged in within the last 20 minutes
}
```

## 🧪 Testing

After making these changes, clear the cache and test your implementation:

```bash
php artisan config:clear
php artisan cache:clear
php artisan serve
```

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📝 License

This project is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
