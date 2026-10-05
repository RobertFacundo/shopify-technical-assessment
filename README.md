### 1. Liquid Output, Money Filters & Sale Logic

**Approach**

I would first compare the product's current price with its `compare_at_price`. The product should only be considered on sale when the `compare_at_price` is greater than the current price.

I would then use Liquid `assign` statements to calculate the savings amount and the percentage saved before rendering them. This keeps the calculation separate from the presentation logic and makes the code easier to read and maintain.

**Liquid**

```liquid
{% assign current_price = product.price %}
{% assign compare_price = product.compare_at_price %}

{% if compare_price > current_price %}
  {% assign savings = compare_price | minus: current_price %}
  {% assign savings_percentage = savings | times: 100.0 | divided_by: compare_price %}

  <div class="sale-savings">
    <span>Save {{ savings | money }}</span>
    <span>{{ savings_percentage | round }}% off</span>
  </div>
{% endif %}
```
The money filter formats the savings amount according to the store's currency and money format.

Consideration

On a product page with multiple variants, sale pricing can vary by variant. In that case, I would use the selected variant's price and compare_at_price rather than relying only on product-level pricing, and update the displayed savings when the selected variant changes.

### 2. Collection Filtering & the Pagination Trap

**Approach**

I would use Shopify's native Storefront Filtering to filter the collection by availability, rather than filtering products manually inside the Liquid loop.

The availability filter can be configured through Shopify's Search & Discovery app and exposed through the collection's `collection.filters`. Once the availability filter is applied, `collection.products` represents the filtered collection, which can then be paginated normally.

I would use Shopify's native pagination and display 12 products per page:

**Liquid**

```liquid
{% paginate collection.products by 12 %}
  <div class="collection-products">
    {% for product in collection.products %}
      {% render 'product-card', product: product %}
    {% endfor %}
  </div>

  {{ paginate | default_pagination }}
{% endpaginate %}
```

**Pagination consideration**

A common mistake is to paginate the collection first and then filter products inside the loop:

```liquid
{% paginate collection.products by 12 %}
  {% for product in collection.products %}
    {% if product.available %}
      ...
    {% endif %}
  {% endfor %}
{% endpaginate %}
```

In this approach, Shopify paginates the original collection before the availability check is performed. If some of those 12 products are out of stock, the page may display fewer than 12 products even though more available products exist on subsequent pages.

Using Shopify's native Storefront Filtering ensures that the availability filter is applied to the collection view before pagination, allowing the native pagination to work correctly with the filtered results.

3. Section Schema with Settings &amp; Repeatable Blocks

### Approach

I would define the configurable options of the section inside the section's `{% schema %}`. These settings are then available in Liquid through `section.settings`.

For the repeatable features, I would define a `feature` block containing an image and a caption. The blocks can then be accessed through `section.blocks`, and each block's individual settings are available through `block.settings`.

### Section Schema

```liquid
{% schema %}
{
  "name": "Features",
  "settings": [
    {
      "type": "text",
      "id": "heading",
      "label": "Heading",
      "default": "Our Features"
    },
    {
      "type": "color",
      "id": "background_color",
      "label": "Background color",
      "default": "#ffffff"
    },
    {
      "type": "range",
      "id": "padding",
      "label": "Padding",
      "min": 0,
      "max": 100,
      "step": 1,
      "unit": "px",
      "default": 40
    }
  ],
  "blocks": [
    {
      "type": "feature",
      "name": "Feature",
      "settings": [
        {
          "type": "image_picker",
          "id": "image",
          "label": "Image"
        },
        {
          "type": "text",
          "id": "caption",
          "label": "Caption"
        }
      ]
    }
  ],
  "presets": [
    {
      "name": "Features"
    }
  ]
}
{% endschema %}
```

### Using the Section Settings

The values defined in the schema can be accessed through `section.settings`.

For example, the heading and background color can be rendered like this:

```liquid
<section
  class="features-section"
  style="
    background-color: {{ section.settings.background_color }};
    padding: {{ section.settings.padding }}px;
  "
>
  <h2>{{ section.settings.heading }}</h2>

  <!-- Feature blocks -->
</section>
```

The `padding` range is defined from `0` to `100` with `px` as its unit, so its value can be output directly as a CSS value:

```liquid
padding: {{ section.settings.padding }}px;
```

### Rendering the Repeatable Blocks

The repeatable blocks are available through `section.blocks`. I would loop through them and access each block's settings through `block.settings`.

```liquid
<div class="features-grid">
  {% for block in section.blocks %}
    <div class="feature" {{ block.shopify_attributes }}>
      {% if block.settings.image != blank %}
        {{ block.settings.image
          | image_url: width: 800
          | image_tag: loading: 'lazy'
        }}
      {% endif %}

      {% if block.settings.caption != blank %}
        <p>{{ block.settings.caption }}</p>
      {% endif %}
    </div>
  {% endfor %}
</div>
```

This allows the merchant to add, remove, and reorder multiple Feature blocks from the Theme Editor while keeping the section-level configuration separate from the repeatable block content.

4. AJAX Cart &amp; Section Rendering API

### Approach

I would intercept the product form submission and use Shopify's Cart API to add the selected variant without refreshing the page.

After the item is successfully added, I would use the Section Rendering API to request the updated header section, extract the cart count from the returned HTML, and replace only that part of the DOM.

I would also keep the JavaScript modular by separating the main operations into small reusable functions rather than putting the entire flow into one large function. This makes the implementation easier to read, maintain, and adapt if the cart UI changes.

### JavaScript

```javascript
const productForm = document.querySelector('[data-product-form]');

productForm?.addEventListener('submit', async (event) => {
  event.preventDefault();

  const formData = new FormData(productForm);
  const variantId = formData.get('id');

  await addToCart(variantId);
});

async function addToCart(variantId) {
  try {
    const response = await fetch('/cart/add.js', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        items: [
          {
            id: Number(variantId),
            quantity: 1
          }
        ]
      })
    });

    if (!response.ok) {
      throw new Error('Unable to add product to cart');
    }

    await updateCartCount();
  } catch (error) {
    console.error(error);
  }
}

async function updateCartCount() {
  const response = await fetch('/?sections=header');

  if (!response.ok) {
    throw new Error('Unable to update cart count');
  }

  const sections = await response.json();
  const parser = new DOMParser();
  const doc = parser.parseFromString(sections.header, 'text/html');

  const newCartCount = doc.querySelector('[data-cart-count]');
  const currentCartCount = document.querySelector('[data-cart-count]');

  if (newCartCount && currentCartCount) {
    currentCartCount.replaceWith(newCartCount);
  }
}
```

The header section would contain a stable wrapper for the cart count, for example:

```liquid
<span data-cart-count>
  {{ cart.item_count }}
</span>
```

### Considerations

* **Cart drawer vs. cart page:** the add-to-cart request can remain the same, but the UI updated after the request would differ. A cart drawer may also need to be opened or refreshed.
* **Error handling:** I would check the response and handle cases such as an invalid or unavailable variant.
* **Preventing duplicate submissions:** the add-to-cart button could be disabled while the request is in progress.
* **Dynamic DOM updates:** if larger parts of the cart UI are replaced, their event listeners may need to be initialized again.
* **Stable selectors:** I would use predictable data attributes for elements that JavaScript needs to update rather than relying on fragile CSS selectors.
* **Maintainability:** I would keep the cart operations separated into reusable functions so the implementation does not become one large block of JavaScript. This also makes it easier to extend the same logic to a cart drawer, quantity updates, or other cart interactions.

### Disclaimer
I have implemented similar AJAX cart and Section Rendering API patterns in a Shopify practice project, including dynamically requesting a section and replacing only the relevant cart markup.

https://github.com/RobertFacundo/shopify-practice
https://lnkd.in/p/d9w-WzJt


# 5. Theme Editor Lifecycle Events

### Approach

I would first make the carousel content reusable through Shopify section blocks, using `section.blocks` in Liquid to render the blocks dynamically.

The issue with the carousel breaking in the Theme Editor is that Shopify can dynamically re-render or replace a section when the merchant adds, removes, or reorders blocks. The page is not necessarily fully reloaded, so the JavaScript initialization that was performed on the original DOM may no longer apply to the new elements.

I would first debug the behavior using browser logs and DOM inspection to confirm when the section is being replaced and whether the carousel is being initialized more than once.

### How I would fix it

I would make the carousel initialization reusable through a function and connect it to Shopify's Theme Editor lifecycle events.

The main events I would use are:

* `shopify:section:load` — triggered when a section is loaded or re-rendered in the Theme Editor.
* `shopify:section:unload` — triggered when a section is removed.
* `shopify:block:select` — triggered when a merchant selects a block.
* `shopify:block:deselect` — triggered when a merchant deselects a block.

For the carousel itself, `section:load` and `section:unload` are the most important events. I would initialize the carousel when the section is loaded and destroy or clean up the previous instance when the section is unloaded.

### Example

```javascript
function initCarousel(section) {
  const carousel = section.querySelector('[data-carousel]');

  if (!carousel) return;

  // Initialize carousel
  console.log('Initializing carousel:', section.dataset.sectionId);

  // Example:
  // section.carousel = new Carousel(carousel);
}

function destroyCarousel(section) {
  if (!section) return;

  console.log('Destroying carousel:', section.dataset.sectionId);

  // Example:
  // section.carousel?.destroy();
  // section.carousel = null;
}

document.addEventListener('shopify:section:load', (event) => {
  initCarousel(event.target);
});

document.addEventListener('shopify:section:unload', (event) => {
  destroyCarousel(event.target);
});
```

The carousel itself would be rendered from the section blocks in Liquid:

```liquid
<div class="carousel" data-carousel>
  {% for block in section.blocks %}
    <div class="carousel__item" {{ block.shopify_attributes }}>
      {{ block.settings.content }}
    </div>
  {% endfor %}
</div>
```

Using `block.shopify_attributes` also allows Shopify to correctly associate the rendered elements with their corresponding Theme Editor blocks.

### Important consideration

I would avoid simply running the initialization code again every time the Theme Editor changes something, because this could create multiple carousel instances and duplicate event listeners.

Instead, I would keep initialization and cleanup separate and make them safe to call repeatedly. This makes the section more reliable both in the Theme Editor and during normal storefront rendering.

# 6. Per-Instance CSS Without Style Bleed

### Approach

I would keep the section's shared CSS for the common layout and styling, while applying the background color selected in the Theme Editor directly to each section instance using Liquid.

Each instance of the section has its own `section.settings`, so the selected background color can be output as an inline style. This ensures that one instance does not affect another.

### Liquid

```liquid
<section
  class="custom-section"
  style="background-color: {{ section.settings.background_color }};"
>
  <div class="custom-section__content">
    {{ section.settings.heading }}
  </div>
</section>
```

The background color would be defined as a section setting:

```liquid
{% schema %}
{
  "name": "Custom Section",
  "settings": [
    {
      "type": "color",
      "id": "background_color",
      "label": "Background color",
      "default": "#ffffff"
    },
    {
      "type": "text",
      "id": "heading",
      "label": "Heading"
    }
  ],
  "presets": [
    {
      "name": "Custom Section"
    }
  ]
}
{% endschema %}
```

### CSS

```css
.custom-section {
  padding: 40px;
}

.custom-section__content {
  max-width: 1200px;
  margin: 0 auto;
}
```

If the merchant adds the section twice and selects a different color for each instance, Shopify will render each instance with its own `section.settings.background_color`.

For example:

```html
<section class="custom-section" style="background-color: #111111;">
```

and:

```html
<section class="custom-section" style="background-color: #eeeeee;">
```

The shared CSS can therefore remain the same, while the instance-specific value is controlled by Liquid.

### Why this prevents style bleed

I would avoid using a shared CSS rule such as `.custom-section { background-color: ...; }` for the configurable color, because that would apply the same value to every instance.

Instead, the background color is scoped to each rendered section instance through its own `section.settings`, so changing one instance in the Theme Editor does not affect the others.

# 7. Integrated Build – AJAX Variant / Size Selector

### Approach

I would render the product's variants and their availability from Liquid and expose the necessary variant data to JavaScript.

When the customer selects a size, JavaScript would find the corresponding variant, update the selected variant ID, update the displayed price, and update the `?variant=` URL parameter without refreshing the page.

For this implementation, I would not need the Section Rendering API because the required variant information is already available on the product page. I can update the relevant DOM elements directly with JavaScript.

### Liquid / HTML

```liquid
<div
  class="product-form"
  data-product-form
>
  <label for="size-selector">Size</label>

  {% assign size_option_index = 0 %}

  {% for option in product.options %}
    {% if option == 'Size' %}
      {% assign size_option_index = forloop.index0 %}
    {% endif %}
  {% endfor %}

  <select id="size-selector" data-size-selector>
    {% for value in product.options_with_values[size_option_index].values %}
      {% assign size_available = false %}

      {% for variant in product.variants %}
        {% if variant.options[size_option_index] == value and variant.available %}
          {% assign size_available = true %}
          {% break %}
        {% endif %}
      {% endfor %}

      <option
        value="{{ value }}"
        {% unless size_available %}disabled{% endunless %}
      >
        {{ value }}
        {% unless size_available %} - Sold out{% endunless %}
      </option>
    {% endfor %}
  </select>

  <div class="product-price" data-product-price>
    {{ product.selected_or_first_available_variant.price | money }}
  </div>

  <input
    type="hidden"
    name="id"
    value="{{ product.selected_or_first_available_variant.id }}"
    data-variant-id
  >

  <script type="application/json" data-variants>
    {{ product.variants | json }}
  </script>
</div>

<script src="{{ 'product-form.js' | asset_url }}" defer></script>
```

The JSON contains the variant IDs, option values, prices, and availability required by the JavaScript.

### JavaScript — `assets/product-form.js`

```javascript
const productForm = document.querySelector('[data-product-form]');

if (productForm) {
  const sizeSelector = productForm.querySelector('[data-size-selector]');
  const priceElement = productForm.querySelector('[data-product-price]');
  const variantInput = productForm.querySelector('[data-variant-id]');
  const variantsElement = productForm.querySelector('[data-variants]');

  const variants = JSON.parse(variantsElement.textContent);

  sizeSelector.addEventListener('change', () => {
    const selectedSize = sizeSelector.value;

    const selectedVariant = variants.find((variant) => {
      return variant.options.includes(selectedSize);
    });

    if (!selectedVariant) return;

    variantInput.value = selectedVariant.id;

    priceElement.textContent = formatMoney(selectedVariant.price);

    const url = new URL(window.location.href);
    url.searchParams.set('variant', selectedVariant.id);

    window.history.replaceState({}, '', url);
  });

  function formatMoney(cents) {
    return new Intl.NumberFormat(undefined, {
      style: 'currency',
      currency: document.documentElement.dataset.currency
    }).format(cents / 100);
  }
}
```

### CSS

```css
.product-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.product-form select {
  width: 100%;
  padding: 10px;
}

.product-form option:disabled {
  color: #999;
}

.product-price {
  font-size: 1.25rem;
  font-weight: 600;
}
```

### Important considerations

The variant ID is the key value connecting the customer's selection to the Shopify variant. Once the appropriate variant is identified, the same ID can be used by the product form when the customer eventually adds the product to the cart.

I would use the History API to update the URL without refreshing the page:

```javascript
window.history.replaceState({}, '', url);
```

`replaceState()` changes the current URL in the browser without triggering a page reload and without creating a new browser history entry. If I wanted each variant selection to create a separate history entry, I could use `history.pushState()` instead.

For a product with multiple options, such as Size and Color, I would extend the matching logic to compare all selected option values rather than only the size.

Using `defer` allows the external JavaScript file to be downloaded without blocking HTML parsing and executed after the document has been parsed, so the product form elements are available when the script runs.

# 8. Metafields: Definition, Rendering & References

### Approach

I would create a product metafield called `Care Instructions` so that merchants can edit the value directly from the Shopify Admin.

I would also create a `Product reference` metafield to associate the current product with another related product. This allows me to access the related product object and then retrieve its own `Care Instructions` metafield.

### 1. Define the Care Instructions metafield

In **Settings → Custom data → Products**, I would create:

* **Name:** Care Instructions
* **Namespace:** `custom`
* **Key:** `care_instructions`
* **Type:** Multi-line text

The merchant can then edit the care instructions from each product in the Shopify Admin.

### 2. Render the metafield on the product page

```liquid id="q2n8wv"
{% assign care_instructions = product.metafields.custom.care_instructions.value %}

{% if care_instructions != blank %}
  <div class="care-instructions">
    <h2>Care Instructions</h2>
    <p>{{ care_instructions }}</p>
  </div>
{% endif %}
```

Using `.value` gives me the actual value stored in the metafield.

### 3. Display Care Instructions from a related product

I would create another product metafield:

* **Name:** Related Product
* **Namespace:** `custom`
* **Key:** `related_product`
* **Type:** Product reference

Then I can access the referenced product through the metafield:

```liquid id="4f7z0k"
{% assign related_product = product.metafields.custom.related_product.value %}

{% if related_product != blank %}
  <div class="related-product-care">
    <h2>Related Product Care Instructions</h2>

    {% if related_product.metafields.custom.care_instructions.value != blank %}
      <p>
        {{ related_product.metafields.custom.care_instructions.value }}
      </p>
    {% endif %}
  </div>
{% endif %}
```

The important part is that a `Product reference` returns a product object through `.value`. I can then access that product's own metafields just like I would access metafields from the current `product`.

This makes the metafields reusable while allowing merchants to manage the content directly from Shopify Admin.


# 9. Metaobjects vs. Metafields

### Why I would use a Metaobject

I would use a Metaobject because `Designer` is a reusable entity with its own set of fields and its own identity.

A metafield is useful for adding a specific piece of data to an existing resource, such as a product. A Metaobject is more appropriate when I need to model a reusable entity that can be referenced by multiple products and also have its own standalone content.

In this case, one Designer could be associated with many products. If the designer's bio, photo, or country changes, I can update the Metaobject once instead of duplicating the same information across multiple products.

### Metaobject configuration

In **Settings → Custom data → Metaobjects**, I would create a `Designer` definition with the following fields:

* **Name** — Single-line text
* **Bio** — Multi-line text
* **Photo** — File/image
* **Country** — Single-line text

I would then create individual Designer entries, such as `John Smith`, `Jane Doe`, etc.

To associate a Designer with products, I would create a product metafield with:

* **Name:** Designer
* **Namespace:** `custom`
* **Key:** `designer`
* **Type:** Metaobject reference
* **Reference:** Designer

This allows multiple products to reference the same Designer entry.

### Rendering the Designer on a product page

```liquid id="a7k3pz"
{% assign designer = product.metafields.custom.designer.value %}

{% if designer != blank %}
  <div class="product-designer">
    {% if designer.photo != blank %}
      {{ designer.photo
        | image_url: width: 400
        | image_tag: loading: 'lazy'
      }}
    {% endif %}

    <h2>{{ designer.name }}</h2>

    {% if designer.bio != blank %}
      <p>{{ designer.bio }}</p>
    {% endif %}

    {% if designer.country != blank %}
      <span>{{ designer.country }}</span>
    {% endif %}
  </div>
{% endif %}
```

The important part is that the product's `designer` metafield is a Metaobject reference. Using `.value` gives me the referenced Designer object, which allows me to access its fields directly.

### Rendering the "Our Designers" page

For the standalone `Our Designers` page, I would retrieve all entries from the Designer Metaobject and loop through them:

```liquid id="v8m2qc"
<div class="designers">
  {% for designer in shop.metaobjects.designer.values %}
    <article class="designer">
      {% if designer.photo != blank %}
        {{ designer.photo
          | image_url: width: 400
          | image_tag: loading: 'lazy'
        }}
      {% endif %}

      <h2>{{ designer.name }}</h2>

      {% if designer.bio != blank %}
        <p>{{ designer.bio }}</p>
      {% endif %}

      {% if designer.country != blank %}
        <span>{{ designer.country }}</span>
      {% endif %}
    </article>
  {% endfor %}
</div>
```

This gives the merchant a reusable Designer entity that can be referenced by multiple products while also allowing all Designer entries to be displayed on a dedicated `Our Designers` page.

# 10. Performance Debugging & Technical Judgment

### Where I would begin

I would start by measuring the storefront before making changes. I would want to identify which part is actually causing the slowdown rather than assuming that every issue has the same impact.

I would use tools such as **Chrome DevTools**, especially the Network and Performance panels, **Lighthouse / PageSpeed Insights**, and the **Shopify Theme Inspector for Chrome** to identify slow Liquid rendering and expensive parts of the theme.

I would also inspect the theme code and the scripts being injected by the six apps.

### What I would prioritize

I would first look at the homepage rendering 50 products with their full variant data. This can generate a large amount of HTML and JavaScript data that the customer may never actually need.

I would reduce the amount of data rendered initially by showing a smaller number of products and using pagination or a "Load more" interaction when appropriate. If variant information is only needed when the customer interacts with a product, I would consider loading that information when it is needed instead of sending the full variant data for every product on the initial page load.

I would then investigate the nested collection loops. I would check whether the same data is being processed repeatedly and remove unnecessary loops or move repeated work outside of product loops where possible.

For example, I would avoid performing expensive collection operations independently for every product if the same information can be obtained once and reused.

### App scripts

I would also investigate the six apps injecting scripts rather than immediately removing them.

Using DevTools, I would check how much JavaScript each app adds, when it loads, and whether the scripts are needed on every page.

If an app is only required on certain pages, I would look for a way to load its functionality only where it is needed. I would also check whether some apps provide overlapping functionality.

### Reusability and maintainability

I would keep the theme code reusable through snippets and components where appropriate, but I would distinguish maintainability from performance.

For example, using a reusable `product-card` snippet is good for maintaining the theme, but if that snippet outputs the full variant data for every product, reusability by itself does not solve the performance problem.

The goal would be to keep the code maintainable while also reducing unnecessary processing and data sent to the browser.

### Balancing functionality and performance

I would avoid removing functionality just to make the storefront faster. Instead, I would first look for ways to provide the same functionality with less initial work.

For example, rather than removing variant selection from product cards, I could avoid sending all variant data for all 50 products and load the additional information only when the customer actually needs it.

After making changes, I would measure the storefront again to verify that the changes actually improved performance and did not negatively affect functionality.

Overall, my process would be:

**Measure → identify the bottleneck → prioritize the highest-impact changes → optimize → measure again.**
