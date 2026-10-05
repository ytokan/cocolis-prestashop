![CI](https://github.com/Cocolis-1/cocolis-prestashop/workflows/CI/badge.svg)

# Cocolis prestashop

This is the repository for our Prestashop module to be able to add [Cocolis.fr](https://www.cocolis.fr/) carrier as a delivery method.

# Fork: PrestaShop 9.1 compatibility

This is an unofficial fork of [Cocolis-1/cocolis-prestashop](https://github.com/Cocolis-1/cocolis-prestashop) (`develop` branch, module version 1.0.10) made to run on **PrestaShop 9.1**. It is tested on PrestaShop 9.1.5 against the Cocolis **sandbox** API. It is not endorsed by Cocolis.

## What changed

- **Installation**: removed hooks that had no matching method, replaced a global constant that no longer exists, and create the carrier price ranges before the zones so the carriers are actually offered at checkout.
- **Uninstallation**: the carriers created by the module are now deleted.
- **Checkout crash**: the `CustomerFormatter` override used an old Symfony interface and made `/commande` fail with a `TypeError` on PrestaShop 9.
- **Order page**: the tracking tab now uses `displayAdminOrderTabLink` / `displayAdminOrderTabContent`. The legacy `displayAdminOrderTabShip` / `displayAdminOrderContentShip` hooks are never fired by PrestaShop 9, so the tab was silently missing.
- **Removed or renamed core APIs**: `Db::executes()`, `Logger::addLog()` and `sizeof()` on a non-array.
- **Configuration**: the webhook button saves the credentials first, an empty password field no longer erases the saved password, and API errors show a message instead of crashing.
- **Ride creation**: sends `content_value` and `external_price`, which the API now requires, sends photos and weight per object, falls back to the customer's mobile number, and logs failures in *Advanced parameters > Logs* instead of swallowing them.
- **Webhooks**: fixed the removed `Logger` call and the SQL injection in the public webhook controller.

The detail of every change, with the cause, is in [docs/PS9-COMPATIBILITY.md](docs/PS9-COMPATIBILITY.md).

## Not confirmed

These points have not been confirmed with Cocolis:

- the meaning of `external_price` (the shipping price paid by the customer is sent for now);
- whether the API accepts `weight` when creating a ride;
- product photos need a shop reachable from the Internet, so they cannot be checked from a local environment;
- only the `ride_published` webhook has been simulated, and the PrestaShop Checkout / PayPal payment flow has not been tested.

## Install

Download the zip of the latest release and upload it in *Modules > Module Manager*. The zip must contain a single `cocolis` folder with its `vendor/` directory.

To build it from the sources:

```
composer install --no-dev --optimize-autoloader
```

Then zip the folder, named `cocolis`.

# Documentation

You can find our documentation on [doc.cocolis.fr](https://doc.cocolis.fr/)

## Test it

You can use docker to test it :

`docker-compose up`
