# Choosing RFID Tags

When using RFID with Inventory, selecting proper RFID tags is crucial for efficiency and reliability.

Before you choose an RFID tag, here's what you need to know:

## RFID "UHF" Tags

The Inventory app is designed to work with RFID UHF (Ultra-High Frequency) tags. These differ from HF (High Frequency) or other types of RFID tags and cannot be used exchangeably. While purchasing RFID tags, please make sure you are choosing the correct type of RFID tags.

{% hint style="info" %}
UHF tags are designed for longer-range reading and are more suitable for asset tracking in various environments.
{% endhint %}

## Choosing the Right RFID Chip for Compatibility

RFID tags are available in various shapes and sizes; however, the features offered by an RFID tag depend on the RFID chip it is equipped with. It is important to ensure that the RFID tag you purchase can meet the desired specifications:

### EPC Memory Bank

You need RFID UHF tags with a writable EPC memory bank of at least 96 bits.

### LOCK Command

During the process of writing an RFID tag, Inventory will also lock the tag using a _non-permanent LOCK command_ (ISO 18000-6C LOCK) to prevent unauthorized or accidental writes to your RFID tags. Since this _non-permanent LOCK command_ is not implemented by all RFID tag chips, selecting RFID tags made with chips that support _non-permanent LOCK commands_ is highly recommended for security and compatibility with Inventory.

{% hint style="info" %}
RFID tags that don't support non-permanent LOCK can still work - although it may appear as a failure while locking the tag while writing to the tag in Inventory, the EPC data can still be successfully written to the tag, making the tag scannable and searchable. However, there are some trade-offs to be aware of:

1. The RFID tag will still be writeable to everyone who has an RFID tag reader. Others may intentionally or unintentionally overwrite the EPC data of your RFID tag, letting you lose track of your item.
2. You may accidentally write to the wrong RFID tag while intending to write or wipe another one. This can cause confusion and chaos - the original item can lose its EPC and become untraceable, and additionally, one item may be mistaken for another if it has the EPC of another item on its RFID tag.
{% endhint %}

{% hint style="warning" %}
Some RFID tag chips may claim that they support the _LOCK_ command, while only _permanent LOCK_ is supported. After permanent _LOCK_, these tags will not be able to be unlocked again, thus becoming permanent read-only tags. These tags are not currently supported by Inventory and will be treated as tags that do not support the _LOCK_ command at all.
{% endhint %}

## Recommended Chips of RFID Tags

The following RFID tag chips have been tested and confirmed to work well with the Inventory app:

* NXP UCODE 8
* Impinj Monza R6-P
* Impinj Monza 730

<details>

<summary>Chips that do not seem to be functioning properly with the Inventory app</summary>

* NXP UCODE 7
* NXP UCODE 9 - it seems that this chip only supports permanent LOCK
* NXP UCODE 9xe

</details>

## On-Metal RFID Tags

**Standard RFID tags might not function properly when attached to metal surfaces, close to metal, or on containers holding liquids.** This is because metal or liquids may detune or absorb the radio wave energy of the antenna on the RFID tag, reducing or nullifying its ability to communicate. To overcome these issues, specialized tags, often called "On-Metal RFID Tags" or "Anti-Metal RFID Tags," are designed to work effectively in such conditions.

Please refer to the [#on-metal-rfid-tags](different-kinds-of-rfid-tags.md#on-metal-rfid-tags "mention") section for more information about common on-metal RFID tags.

##
