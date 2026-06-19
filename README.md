# JM WIFI Billing System

A complete web-based billing system for an internet service provider, with
live MikroTik PPPoE/IPoE integration and PayMongo (GCash) payment automation.

Built on **Laravel 11** + **MySQL**, deployable on any standard LAMP-style VPS.

## What's Included

- **3 user roles**: Admin, Staff, Client — each with their own login, guard, and dashboard
- **MikroTik RouterOS API integration**: PPPoE secret/profile management, IPoE/DHCP lease + address-list management, live online/offline detection, full audit log of every router action
- **Automatic billing lifecycle**: scheduled job marks clients expired and suspends them on the router; payment automatically restores and reconnects them
- **PayMongo GCash checkout + webhook**, with signature verification and idempotent payment processing
- **Manual payment recording** and **client-submitted proof-of-payment** workflow for cash/manual GCash transfers
- **Client self-service portal**: pay bills, advance payment, payment history, downloadable/printable receipts
- **Public expired-account portal** that redirects disconnected clients straight to payment
- **Admin dashboard**: live stats, router status, monthly collection, VPS resource usage
- **Google Maps client location view** with status-colored pins (green/red/gray) and a location picker
- **Full MySQL schema** for clients, plans, routers, payments, invoices, MikroTik logs, system settings, and client locations

## Quick Start

See **[docs/INSTALL.md](docs/INSTALL.md)** for the full step-by-step VPS deployment guide (PHP/MySQL/Nginx setup, MikroTik API setup, PayMongo webhook setup, queue worker, cron scheduler).

In short:

```bash
composer install --optimize-autoloader --no-dev
cp .env.example .env
php artisan key:generate
# edit .env: DB credentials, PAYMONGO_*, GOOGLE_MAPS_API_KEY, APP_URL
php artisan migrate
php artisan db:seed
php artisan storage:link
```

**Default admin login** (change immediately after first login):
- Email: `admin@jmwifi.local`
- Password: `ChangeMe!2026`

## Testing

See **[docs/TESTING.md](docs/TESTING.md)** for a full checklist covering MikroTik connectivity, PPPoE/IPoE suspend-restore cycles, PayMongo webhook verification, manual payments, and dashboard accuracy.

## Project Structure

```
app/
  Console/Commands/      Scheduled artisan commands (sync, expire-check, warnings)
  Http/Controllers/      Admin, Staff, Client, Auth, Public, Webhooks
  Http/Middleware/       Role guards (IsAdmin, IsAdminOrStaff, IsClient)
  Jobs/                  Queued MikroTik actions (reconnect, suspend, sync)
  Models/                Eloquent models for all 9+ tables
  Notifications/         Admin + client notifications
  Services/MikroTik/     RouterOS API connection, PPPoE manager, IPoE manager
  Services/PayMongo/     PayMongo API client, payment processor
database/
  migrations/            All table definitions
  seeders/                Default admin/staff/plans/settings
resources/views/         Blade templates (admin, staff, client, public, auth)
routes/
  web.php                All application routes
  webhooks.php           PayMongo webhook (CSRF-exempt)
  console.php            Scheduled task definitions
docs/
  INSTALL.md             Full VPS deployment guide
  TESTING.md             Testing checklist
```

## Key Design Notes

- **Clients authenticate separately from Admin/Staff** (`client` guard vs `web` guard), so a billing-system breach on one side doesn't expose the other.
- **Router credentials are encrypted at rest** in the database.
- **Every MikroTik action is logged** to `mikrotik_logs` — when a client says "I paid but I'm still not connected," this table tells you exactly what happened (or didn't).
- **PayMongo webhook signature is verified** before any payload is trusted; payment processing is idempotent so a retried webhook can't double-extend an account.
- **MikroTik actions run as queued jobs** with retry/backoff, so a momentarily unreachable router doesn't fail silently — it retries automatically and lands in `failed_jobs` for admin review if it keeps failing.
