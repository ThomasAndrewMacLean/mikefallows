---
draft: false
title: Implement a low stock notice in a Shopify theme
description: >-
  A simple way to display a notice when the selected variant has a stock
  quantity below a specified value.
date: 2023-10-19T23:00:00.000Z
tags:
  - shopify
  - javascript
layout: post
---

I recently needed to implement a small feature on a Shopify theme to display a low stock notice.

## Requirements

This was a quick task that simply needed to display a low stock notice when the selected variant was below a certain threshold, and reflect the change when another variant was selected. For example, given the following scenario:
- low stock threshold is 5
- product has two variants: _Standard_ and _Deluxe_
- _Standard_ has 20 items in stock
- _Deluxe_ has 3 items in stock

The notice should only show when the _Deluxe_ item is selected and it should include the number of units left in stock, i.e. "3 in stock".

## Demo

Here's a demo (try selecting _Deluxe_):

{% include '_includes/demos/low-stock.njk' %}

## Considerations

Because this theme had the option to display variants as a `select` dropdown, or as `radio` options that acted like buttons to select the variant, I decided to look for a way I could make this feature generic enough to work with both. I noticed both versions would dynamically update a hidden variant input to reflect the chosen variant's ID. This made it easy to observe changes to that ID and to know when to update the notice and identify the currently selected variant.

## The code

As I wanted to enable this as a section (so that it could be used conditionally on different templates), I could use Liquid to store the variant stock quantities in a JavaScript variable.

{% raw %}
```js
const inventory = [
  {%- for v in product.variants %}
    { id: '{{ v.id }}', quantity: '{{ v.inventory_quantity }}' },
  {%- endfor %}
];
```
{% endraw %}

Assuming that the HTML structure of the page is something that could be boiled down to a simplified structure like this:

```html
<div class="product">
  <form>
    <input type="hidden" class="product-variant-id" value="123">
    <select name="id" class="quantity">
      <option value="123">Standard</option>
      <option value="456">Deluxe</option>
    </select>
  </form>
</div>
```

Then the script would look like this:

{% raw %}
```html
<script>
  window.addEventListener('DOMContentLoaded', () => {

    // Initial config
    const threshold = 5;
    const productSelector = '.product';
    const quantitySelector = '.quantity';
    const hiddenVariantIdInputSelector = '.product-variant-id';
    const initialVariantId = '{{ product.selected_or_first_available_variant.id }}';

    // Store stock quantities
    const inventory = [
      {%- for v in product.variants %}
        { id: {{ v.id | json }}, quantity: {{ v.inventory_quantity | json }} },
      {%- endfor %}
    ];

    // Find the root product element
    const productEl = document.querySelector(productSelector);
    if (!productEl) return;

    // Find the quantity selector (we will append our low stock notice container to this)
    const quantityEl = productEl.querySelector(quantitySelector);
    if (!quantityEl) return;

    // Append the notice container
    quantityEl.insertAdjacentHTML('afterend', `<div class="stock-notice"></div>`);

    // Method to update the content of the notice container
    // if the stock is not above the threshold
    function updateMessage(variantId) {
      const stockNoticeEl = productEl.querySelector('.stock-notice');
      if (!stockNoticeEl) return;
      let message = '';
      const variant = inventory.find(i => i.id === variantId);
      if (variant && (variant.quantity > 0 && variant.quantity <= threshold)) {
        message = `${variant.quantity} in stock`;
      }
      stockNoticeEl.textContent = message;
    }

    // Update the message on first page load
    updateMessage(initialVariantId);

    // Find the input that is updated with the selected id
    const hiddenVariantIdInputEl = productEl.querySelector(hiddenVariantIdInputSelector);
    if (!hiddenVariantIdInputEl) return;

    // Watch the input for changes and update the message in response
    hiddenVariantIdInputEl.addEventListener('change', (e) => updateMessage(e.target.value));

  });
</script>
```
{% endraw %}

## Customisation

You could add the inventory threshold as a section setting. That would also allow you to populate the threshold through a value stored on the product as a metafield 🙌