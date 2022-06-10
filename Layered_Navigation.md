# Anatomy of Magento 2: Layered Navigation

## Points for investigation

* Generate high-level overview of all the main components
* How are the total item counts for each option generated?
* How is the catalog updated to display the matching products?
* How are options rendered?
* What are the customisation points?
* What events are dispatched?
* What observers listen on the process?
* What plugins are used?
* What effect do the options in the Manage Attributes section of the admin have?
* Create flowcharts
* How does the logic differ between the catalog and search results pages?
* How does this work with caching?

## Architecture

## Key concepts

* `\Magento\Catalog\Model\Layer\Resolver` Creates a layer object of the given type ('catalog' or 'search')
* `\Magento\Catalog\Model\Layer` Layer view model
* `\Magento\Catalog\Model\Layer\State` State model for the layer. Manages which filters are applied to the layer.
* `\Magento\Catalog\Model\Layer\Filter\DataProvider\Category` Provides logic to load the category and use it as a filter
* `\Magento\Catalog\Model\Layer\Filter\DataProvider\Decimal` Provides logic to use decimal values (excluding prices) as a filter (e.g. Managing range values)
* `\Magento\Catalog\Model\Layer\Filter\DataProvider\Price` Provides logic to use prices as a filter (e.g. Managing ranges, price steps, applying config settings)
* `\Magento\Catalog\Model\Layer\FilterList` Manages the types of filters that can be used. Core filter types include 'category', 'attribute', 'price' and decimal'. Filters can be added or replaced by adding elements to the `\Magento\Catalog\Model\Layer\FilterList::__construct(array filters)` argument.
* `\Magento\Catalog\Model\Layer\Category\FilterableAttributeList` Returns collection of attributes that can be filtered by layered navigation (Defined by setting the attribute property `is_filterable` to true)
* `\Magento\LayeredNavigation\Block\Navigation` Contains logic for starting rendering process of current layer
* `\Magento\LayeredNavigation\Block\Navigation\State` Manages state of the current layer
* `\Magento\Framework\Search\Response\Aggregation` A collection of buckets for a layer
* `\Magento\Framework\Search\Response\Bucket` A container for a filter and it's values

## Code safari

First load of an anchor category, before selecting any options to filter by.

* The controller for category pages, `\Magento\Catalog\Controller\Category\View`, checks if layered navigation is enabled for this category and if so, adds custom handles to the layout.
* The default block for adding layered navigation to a category is `catalog.leftnav`.
* Both `Magento_Catalog` and `Magento_LayeredNavigation` define blocks with this name, in `/vendor/magento/module-catalog/view/frontend/layout/catalog_category_view_type_default.xml` and `/vendor/magento/module-layered-navigation/view/frontend/layout/catalog_category_view_type_layered.xml` respectively.
    * Question: Given these blocks are added to the same container (`sidebar.main`), why don't these blocks conflict?
        * The class defined by both blocks is different
* The block class for `catalog.leftnav` in `catalog_category_view_type_default.xml` resolves to a virtual type, which extends from `\Magento\LayeredNavigation\Block\Navigation`.

### Filtering the collection
* In `\Magento\LayeredNavigation\Block\Navigation::_prepareLayout`, applicable filters for the current layer are retrieved and then applied to the current request.
    * `Magento\Catalog\Model\Layer\Category\FilterableAttributeList`: Gets collection of filterable attributes
        * `\Magento\Catalog\Model\Layer\FilterList`: Creates the filters.
    * `\Magento\CatalogSearch\Model\Layer\Filter\Category`: Executes logic specific to category filters. Gets the category ID from the request and filters the collection of the current layer by it.
        * `\Magento\Catalog\Model\Layer\Filter\DataProvider\Category`: Returns the current category, loading it if necessary, then stores it in registry (`current_category_filter`).
            * `\Magento\CatalogSearch\Model\Layer\Category\ItemCollectionProvider:` Returns filtered collection of items for the category filter type
            * `\Magento\CatalogInventory\Model\Plugin\Layer::beforePrepareProductCollection`: Filters collection by stock status (defaults to hiding out-of-stock items)
                * `\Magento\Catalog\Model\Layer\Category\CollectionFilter`: Adds default product attributes, minimal price, final price, 'tax percents', URL rewrites (for this category) to collection and filters by product visibility
        * `\Magento\Catalog\Model\Layer::$_productCollections`: The filtered collection is then stored to the `\Magento\Catalog\Model\Layer::$_productCollections` property
    * This process of filtering the collection by filters passed in the request is repeated for each filter
    * `\Magento\Catalog\Model\Layer::apply`: The filters are then applied to the current layer. Gets all the filters applicable to the current state and creates a unique key for the state.
* `\Magento\LayeredNavigation\Block\Navigation::_prepareLayout` delegates to the parent `_prepareLayout`, which eventually dispatches the `core_layout_block_create_after` event.

### Displaying it
* `\Magento\Catalog\Block\Product\ListProduct::initializeProductCollection`:
    * Gets the filtered collection from the layer block (`\Magento\Catalog\Model\Layer\Category` in this case)
    * Sorts the collection by applicable sort orders (Price, Position, etc)
    * Sets the collection to the toolbar block (to ensure that pagination, product counts are correct)
* `/vendor/magento/module-catalog/view/frontend/templates/product/list.phtml` gets product collection
    * Outputs toolbar
    * Outputs product grid (or list)
* `/vendor/magento/theme-frontend-luma/Magento_LayeredNavigation/templates/layer/view.phtml`:
    * Checks whether block can be output
    * Outputs number of active filters
    * Calls child block 'state' of type `\Magento\LayeredNavigation\Block\Navigation\State`
        * `/vendor/magento/theme-frontend-luma/Magento_LayeredNavigation/templates/layer/state.phtml` Renders currently applied filters
    * Outputs list of filters
        * Each filter has it's own renderer block class and accompanying template
            * Plugin `\Magento\Swatches\Model\Plugin\FilterRenderer::aroundRender` checks if this filter attribute is a swatch attribute and if so, renders a custom block
        * `\Magento\LayeredNavigation\Block\Navigation\FilterRenderer::render` calls the `toHtml` method to render each filter
        * The default template for each filter is `/vendor/magento/module-layered-navigation/view/frontend/templates/layer/filter.phtml`
        * The block class is `\Magento\Catalog\Model\Layer\Filter\Item`
            * Outputs the different options for each filter attribute
            * Outputs the count of each option for each filter attribute

### How product counts are generated

TODO: Finish this section

Magento 2 uses the CatalogSearch module in order to generate the counts of each product.

* `\Magento\Catalog\Block\Product\ListProduct::initializeProductCollection` dispatches event `catalog_block_product_list_collection`
    * This is observed by `\Magento\Review\Observer\CatalogBlockProductCollectionBeforeToHtmlObserver`, which loads the collection
    * In `\Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection::_renderFiltersBefore`, search criteria is created and passed to the search instance, `\Magento\Search\Api\SearchInterface`, which in turn queries the configured search engine, `\Magento\Search\Model\SearchEngine`

#### For filter type 'category'
* `\Magento\LayeredNavigation\Block\Navigation::canShowBlock` triggers a process which initialises all filter options
    * `\Magento\Catalog\Model\Layer\Category\AvailabilityFlag::isEnabled` tries to load all filters
    * `\Magento\Catalog\Model\Layer\Category\AvailabilityFlag::canShowOptions` cycles through all filters and loads the item count for each one.
        * That loop eventually calls `\Magento\Catalog\Model\Layer\Filter\AbstractFilter::_initItems`
        * `\Magento\CatalogSearch\Model\Layer\Filter\Category::_getItemsData`: Gets collection from layer model, calls `getFacetedData`
            * `\Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection::getFacetedData`: Renders filters of layer product collection; Returns faceted data from faceted search result for filter
                * Gets aggregations from searchResult; Loops through buckets in aggregations
* `\Magento\Catalog\Block\Product\ListProduct::initializeProductCollection` dispatches event `catalog_block_product_list_collection`
    * This is observed by `\Magento\Review\Observer\CatalogBlockProductCollectionBeforeToHtmlObserver`, which loads the collection
    * In `\Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection::_renderFiltersBefore`, search criteria is created and passed to the search instance, `\Magento\Search\Api\SearchInterface`, which in turn queries the configured search engine, `\Magento\Search\Model\SearchEngine`

#### For filter type 'attribute'
* `\Magento\LayeredNavigation\Block\Navigation::canShowBlock` triggers a process which initialises all filter options
    * `\Magento\Catalog\Model\Layer\Category\AvailabilityFlag::isEnabled` tries to load all filters
    * `\Magento\Catalog\Model\Layer\Category\AvailabilityFlag::canShowOptions` cycles through all filters and loads the item count for each one.
        * That loop eventually calls `\Magento\Catalog\Model\Layer\Filter\AbstractFilter::_initItems`
        * `\Magento\CatalogSearch\Model\Layer\Filter\Category::_getItemsData`: Gets collection from layer model, calls `getFacetedData`
            * `\Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection::getFacetedData`: Renders filters of layer product collection; Returns faceted data from faceted search result for filter
                * Gets aggregations from searchResult; Gets buckets for this attribute from aggregations
                * _Buckets_ contains _Metrics_. Metrics are a count of how many times the filter option (i.e. attribute value) appeared in the search result. These are the counts which are eventually displayed beside each option in the layered navigation block.
            * Gets all frontend attribute options and generates filter options/items from them
                * `\Magento\CatalogSearch\Model\Layer\Filter\Attribute::buildOptionData` Detects whether this option reduces the number of products displayed. If not, the filter option is not rendered. If so, a new filter item is created and added to an internal array. This array is built up to contain the frontend label, attribute value and product count for each filter option of this attribute.

#### For filter type 'price'
* `/vendor/magento/theme-frontend-luma/Magento_LayeredNavigation/templates/layer/view.phtml:33` eventually calls
    * `\Magento\Catalog\Model\Layer\Filter\AbstractFilter::_initItems`
        * `\Magento\CatalogSearch\Model\Layer\Filter\Category::_getItemsData`: Gets collection from layer model, calls `getFacetedData`
            * `\Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection::getFacetedData`: Renders filters of layer product collection; Returns faceted data from faceted search result for filter
                * Gets aggregations from searchResult; Gets buckets for this attribute from aggregations
                * _Buckets_ contains _Metrics_. Metrics are a count of how many times the filter option (i.e. attribute value) appeared in the search result. These are the counts which are eventually displayed beside each option in the layered navigation block.
            * Checks that there is more than one 'facet' (attribute option) (in order to generate a valid range)
                * Each 'facet' contains a value representing the price steps (e.g. 100-200), a count of how many products fit this facet and 'from' and 'to' values representing the upper and lower bounds of the range.



`\Magento\Catalog\Model\Layer\Resolver`:
* Is used to determine the type of layer to use for this request: 'category' or 'search'.
* The types are defined in `\Magento\Catalog\Model\Layer\Resolver`
* The types are added to a `layersPool` argument which is populated in `/vendor/magento/module-catalog/etc/di.xml:490` and injected by `di`.
* Which opens the possibility of creating a custom type?
* The resolver returns a layer object of the configured type
* Both of the default types extend from `\Magento\Catalog\Model\Layer`.

* Rendering starts in `\Magento\LayeredNavigation\Block\Navigation::_prepareLayout`


## How does the logic differ between the catalog and search results pages?

There are separate layers for both:
`/vendor/magento/module-catalog/etc/di.xml:490`