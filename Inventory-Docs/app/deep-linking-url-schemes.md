---
description: >-
  With deep linking, you can open an item or page directly in Inventory from a
  web page, iOS Shortcut, or by scanning a QR code.
---

# Deep Linking (URL Schemes)

The following URLs are supported in the Inventory app:

| Type                                            | Identifier                                                                 | URL                                        |
| ----------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------ |
| **Item**                                        | <p>Individual Asset Reference<br>(e.g.: <code>1234.123456.0000</code>)</p> | `z-inv://iar/<individual_asset_reference>` |
|                                                 | [ID](deep-linking-url-schemes.md#how-to-get-the-id-of-an-item)             | `z-inv://item/<item_id>`                   |
| **Collection**                                  | [ID](deep-linking-url-schemes.md#how-to-get-the-id-of-an-item)             | `z-inv://collection/<collection_id>`       |
| **Nothing (only opens the app but do nothing)** | -                                                                          | `z-inv://nothing`                          |



<details>

<summary>How to get the ID of an item?</summary>

Navigate to the item, press the three dots button on the top-right of the screen, then select "Copy Item ID".\
<img src="https://hackmd.io/_uploads/BkDaOtYCh.png" alt="" data-size="original">

</details>

### Using Multiple Profiles

If you have multiple profiles and wish to switch to a specific profile with deep links, add `?p=<profile_id>` or `?c=<config_uuid>` at the end of the URL. For example, `z-inv://iar/<individual_asset_reference>?p=a1b2c3d4`.

<details>

<summary>How to get the Profile ID?</summary>

More (on the bottom tab bar) → Profile Icon (on the top-right) → Copy Current Profile ID.

</details>

<details>

<summary>How to get the Config UUID?</summary>

More (on the bottom tab bar) → Settings → Configuration → Configuration UUID.

</details>

***

{% hint style="info" %}
**URL Schemes for the Nightly and Dev versions of the app (iOS only)**

* Use `zn-inv://` for the Nightly version of the app.
* Use `zd-inv://` for the Dev version of the app.
{% endhint %}
