# Troubleshooting

## API 500 "Invalid key supplied" (LogicException in CryptKey)

If `/api/profile` or other API routes return 500 with:

```json
{"error":"LogicException","message":"Invalid key supplied","file":".../CryptKey.php","line":67}
```

Laravel Passport’s encryption keys are missing. Generate them and (if needed) install clients:

**Local / single server:**
```bash
cd beta   # or your app directory
php artisan passport:keys
# If this is a fresh install and you also need OAuth clients:
php artisan passport:install
```

**Docker (beta/production):** run inside the app container so keys are created in the container’s `storage/` (and persist if that directory is a volume):

```bash
docker compose exec app php artisan passport:keys
# Optional, for fresh install:
docker compose exec app php artisan passport:install
```

Keys are written to `storage/oauth-private.key` and `storage/oauth-public.key`. Alternatively set `PASSPORT_PRIVATE_KEY` and `PASSPORT_PUBLIC_KEY` in `.env` (full PEM contents; use `\n` for newlines) and no key files are needed.

## `composer install` (Windows / local PHP)

If `composer install` fails with missing extensions or PHP version errors:

1. **Enable required PHP extensions**  
   Open your `php.ini` (run `php --ini` to see its path, e.g. `C:\tools\php85\php.ini`). Ensure these lines are present and uncommented:
   ```ini
   extension=fileinfo
   extension=sodium
   ```
   Restart the terminal and run `composer install` again.

2. **PHP 8.5 and "does not satisfy" version errors**  
   The lock file was generated for PHP 8.0–8.4. If you use PHP 8.5:
   - Prefer switching to **PHP 8.2 or 8.4** for this project (e.g. via your version manager), or
   - Run: `composer install --ignore-platform-reqs`  
   Then still enable `fileinfo` and `sodium` (step 1) so the app works at runtime (Passport/JWT and uploads need them).

3. **PHP 8.2+ required**  
   This project uses Laravel 12 and requires PHP 8.2 or higher.

## 413 when uploading profile photo (not using Docker)

If you use `php artisan serve` and get "413 Request Entity Too Large", increase PHP limits in your `php.ini`: `upload_max_filesize` and `post_max_size` (e.g. 10M). Run `php --ini` to find the loaded `php.ini`.

## Unit Test

### run PHPUnit

```bash
# run PHPUnit all test cases
vendor/bin/phpunit
# or Feature test only
vendor/bin/phpunit --testsuite Feature
```

### Running tests (Docker)

With the app stack running (`docker compose up -d`), run PHPUnit inside the app container:

```bash
docker compose exec app php vendor/bin/phpunit
```

Use this when PHP is not on your PATH (e.g. on Windows) or to match the container environment.

### Code Coverage Report

```bash
# reports is a directory name
vendor/bin/phpunit --coverage-html reports/
```
A `reports` directory has been created for code coverage report. Open the dashboard.html.
