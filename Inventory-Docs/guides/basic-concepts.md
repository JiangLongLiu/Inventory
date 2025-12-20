# Basic Concepts

## Item and Collection

An **Item** in Inventory represents an individual asset you own or manage. Each item can be detailed with information such as purchase date, price, supplier, expiry date, and any other relevant details you wish to track.

**Items are categorized into Collections**, which are like broad categories or folders. Collections help you organize your items into meaningful groups. For example, you might have Collections named "Books," "Tools," or "Office Supplies."

Every item must belong to a collection, so you’ll need to create the necessary collections before adding items.

### Item Types

Items in Inventory can have different types, each tailored to suit various needs:

1. **Normal Item**:\
   This is a standard asset, like an office chair, a book, or a hammer.
2.  **Item with Parts**:

    Some items come with additional parts. For instance, a label printer might have a power adaptor and a USB cable. This type allows you to nest these parts under their main item, allowing each part to be tracked individually.
3.  **Container**:

    Imagine a toolbox or a drawer. This type lets you nest items that will usually be stored within another item, perfect for keeping track of grouped assets.
4.  **Consumable**:

    Tailored for regularly used and replenished items, such as boxes or bags of batteries, water filters, light bulbs, paper towels, pieces of electronic modules, or rolls of 3D printing filaments. It enables monitoring of quantities and setting minimum stock levels, assisting in timely restocking and inventory control.

## Individual Asset Reference (IAR)

The Individual Asset Reference (IAR) is an optional and unique identifier of assets within the Inventory app. It provides a convenient way to uniquely identify an item, which can be used within QR codes, barcodes, or RFID tags.

An item must have an IAR to be RFID-tagged, as the IAR forms a significant part of the Electronic Product Code (EPC) encoded into an item's RFID tag.

### Structure of IAR

By default, the IAR consists of three parts:

* **Collection Reference Number (4 digits):** Derived from the item's Collection Reference Number, this 4-digit code categorizes the item within a specific collection.
* **Item Reference Number (6 digits):** A unique code assigned to each item type, which can be manually set or randomly generated.
* **Serial Number (4 digits):** Identifies individual units within the same item type. With 0 by default, you can use the serial number freely to:
  * Add more units of the same model with a shared Item Reference Number using an incremental serial number.
  * Consistent use of the Item Reference Number for accessories or parts of a specific asset, differentiated only by their serial number.
  * Use the same Item Reference Number for items that should be stored in a specific container, such as a toolbox.

Example: `1234.123456.0000`

```
1234.123456.0000
╰┬─╯ ╰┬───╯ ╰┬─╯
 │    │     Serial
 │    │
 │   Item Reference Number
 │
Collection Reference Number
```

<details>

<summary>Design Rationale</summary>

* **Enhanced Organization and Identification:** The combination of collection and item IDs makes it straightforward to identify what an item is and where it belongs at a glance.

<!---->

* **Efficient RFID Scanning:** When checking specific items, the structured IAR facilitates efficient RFID scanning by allowing the use of a narrower EPC filter. For example, when checking items within a specific toolbox, which all have the IAR starting with `1234.123456`, the app can use an EPC filter such as `1234.123456.xxxx`, to ensure the RFID reader only detects relevant tags. This prevents interference from unrelated tags, streamlining the scanning process.

</details>

