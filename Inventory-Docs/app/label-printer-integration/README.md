# Label Printer Integration

In theory, the Inventory app supports all label printer hardware and software that lets you use HTTP API to print tags. For example, [Label LIVE](https://label.live/).

Label printer integration is done by defining a configuration object that contains the following properties:&#x20;

* The `options` which is adjustable while printing the tag(s).&#x20;
* A `getLabel` function that returns the label's data for a given item.
* A `print` function that receives an array of label data and calls APIs to print them.
* Optionally, a `getPreview` function to return an image source for a given label image.

## Example Integrations

* [Label LIVE](label-live.md)&#x20;

