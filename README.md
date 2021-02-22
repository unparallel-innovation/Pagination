Kurounin:Pagination
=================

This package allows you to paginate the subscriptions over meteor's collections. It can be used in a Blaze template or in ReactJS.


Features
--------

+ **Incremental subscriptions**. Downloads only what is needed, not the entire collection at once. Suitable for large datasets.
+ **Instant changes propagation**. Any document changes are instantly propagated, thanks to light-weight modifications of subscription mechanism.
+ **Easy integration**. The package works out of the box. Page changes are triggered by a single reactive dictionary variable.
+ **Multiple collections per page**. Each Pagination instance runs independently. You can even create multiple paginations for one collection on a single page.
+ **Bootstrap 3 and 4 compatible navigation template**. Blaze template for a bootstrap 3 and 4 styled paginator.
+ **Bootstrap 3 and 4 compatible navigation react class**. ReactJS class for a bootstrap 3 and 4 styled paginator.
+ **Supports aggregation queries on publication**. Support added using [tunguska:reactive-aggregate](https://atmospherejs.com/tunguska/reactive-aggregate) package

# Installation
```meteor add kurounin:pagination```

**For Blaze paginator install [kurounin:pagination-blaze](https://atmospherejs.com/kurounin/pagination-blaze) package**
```meteor add kurounin:pagination```

**For ReactJS paginator in Meteor 1.2 install [kurounin:pagination-reactjs](https://atmospherejs.com/kurounin/pagination-reactjs) package**
```meteor add kurounin:pagination-reactjs```

**For ReactJS paginator in Meteor 1.3+ install [react-bootstrap-pagination](https://www.npmjs.com/package/react-bootstrap-pagination) npm package**
```npm i react-bootstrap-pagination```

# Usage

In your collections file (e.g. lib/collections.js):
```js
MyCollection = new Meteor.Collection('myCollectionName');
```

In your publications file (e.g. server/publications.js):
```js
import { publishPagination } from 'meteor/kurounin:pagination';

publishPagination(MyCollection);
```

Optionally you can provide a set of filters on the server-side, a dynamic filters which can not be overridden and an aggregation pipeline  
There's also the option of providing a transformation filter function to validate the client filters (e.g. server/publications.js):
```js
publishPagination(MyCollection, {
    filters: {is_published: true},
    dynamic_filters: function () {
        return {user_id: this.userId};
    },
    //Aggregation support is useful when related sets of data from multiple collections are needed on a given page
    aggregate: {
        pipeline: (props)=>([
        	//pipeline stages goes here
        ]),
        //By default the aggregate only re-runs when collection "MyCollection" changes
        //To trigger a re-run when other collections changes you must define an observer for each additional collection
        observers:[
            AnotherCollection.find()
        ],
        //A schema is required for ObjectIDs fields, a detailed example is given on Aggregation section
        schema:new SimpleSchema(/*...*/)
    },
    transform_filters: function (filters, options) {
        // called after filters & dynamic_filters
        allowedKeys = ['_id', 'title'];

        const modifiedFilters = [];

        // filters is an array of the provided filters (client side filters & server side filters)
        for (let i = 0; i < filters.length; i++) {
            modifiedFilters[i] =  _.extend(
                _.pick(filters[i], allowedKeys),
                {user_id: this.userId}
            );
        }

        return modifiedFilters;
    },
    transform_options: function (filters, options) {
        const fields = { name: 1, email: 1 }
        if (Roles.userIsInRole(this.userId, 'admin')) {
            fields.deleted = 1;
        }
        options.fields = _.extend(fields, options.fields);
        return options;
    }
});

```



For Blaze template
--------------------------------------------------
In your template file (e.g. client/views/mylist.html):
```html
<template name="myList">
    <div>
        {{#if isReady}}
            <ul>
              {{#each documents}}
                  <li>Document #{{_id}}</li>
              {{/each}}
            </ul>
            {{> defaultBootstrapPaginator pagination=templatePagination limit=10 containerClass="text-center" onClick=clickEvent}}
        {{else}}
            Loading...
        {{/if}}
    </div>
</template>
```
**[kurounin:pagination-blaze](https://atmospherejs.com/kurounin/pagination-blaze) package is needed for paginator**


In your template javascript file (e.g. client/scripts/mylist.js):
```js
Template.myList.onCreated(function () {
    this.pagination = new Meteor.Pagination(MyCollection, {
        sort: {
            _id: -1
        }
    });
});

Template.myList.helpers({
    isReady: function () {
        return Template.instance().pagination.ready();
    },
    templatePagination: function () {
        return Template.instance().pagination;
    },
    documents: function () {
        return Template.instance().pagination.getPage();
    },
    // optional helper used to return a callback that should be executed before changing the page
    clickEvent: function() {
        return function(e, templateInstance, clickedPage) {
            e.preventDefault();
            console.log('Changing page from ', templateInstance.data.pagination.currentPage(), ' to ', clickedPage);
        };
    }
});
```

For ReactJS template
--------------------------------------------------
In your view file (e.g. client/views/mylist.jsx):
```html
MyListPage = React.createClass({
    mixins: [ReactMeteorData],

    pagination: new Meteor.Pagination(MyCollection),

    getMeteorData: function() {
        return {
            documents: this.pagination.getPage(),
            ready: this.pagination.ready()
        };
    },

    renderDocument: function(document) {
        return (
            <li>Document #{document._id}    </li>
        );
    },

    render: function() {
        if (!this.pagination.ready()) {
            return (
                <div>Loading...</div>
            );
        }

        return (
            <div>
                <ul>
                    {this.data.documents.map(this.renderDocument)}
                </ul>
                <DefaultBootstrapPaginator
                    pagination={this.pagination}
                    limit={10}
                    containerClass="text-center"
                    />
            </div>
        );
    }
});
```
**For Meteor 1.2 [kurounin:pagination-reactjs](https://atmospherejs.com/kurounin/pagination-reactjs) package is needed for paginator**

**For Meteor 1.3+ [react-bootstrap-pagination](https://www.npmjs.com/package/react-bootstrap-pagination) npm package is needed for paginator**

#Aggregation

This example show how o publish relational data using an aggregate with a `$lookup` pipeline stage

**note:** When using ObjectIDs since Meteor doesn't have native support for aggregation, any ObjectID field (different than _id) sent to the client will not be displayed correctly

To ensure that those fields are displayed correctly a schema to ObjectID fields must be defined

To enable this support the npm packages ``simpl-schema`` and ``lodash`` or ``lodash-es`` must be installed

``meteor npm install simpl-schema``

``meteor npm install lodash-es`` or ``meteor npm install lodash``

On Server
--------------------------------------------------

```js
publishPagination(MyCollection, {
    aggregate: {
        pipeline: (props)=>(
            [
                {
                    $lookup:{
                        from: "myRelatedCollection",
                        localField: props.lookupField,
                        foreignField: "_id",
                        as:"_relatedDocs"
                    }
                }
            ]
        ),
        //By default the aggregate only re-runs when collection "MyCollection" changes
        //To trigger a re-run when collection "MyRelatedCollection" changes you must define an observer
        observers:[
            MyRelatedCollection.find()
        ],
        //If a schema is not defiend MyCollection.schema will be used
        schema:new SimpleSchema({
            "_relatedDocs": {type: Array},
            "_relatedDocs.$":{type: Object},
            "_relatedDocs.$._id":{type: Mongo.ObjectID}
        })
    }
})

```

On Client
--------------------------------------------------
```js
new Meteor.Pagination(MyCollection,{
    aggregateProps: {lookupField: "relatedIds"}
})
```


# Demo project
For an example on how this package can be implemented check [the pagination example project](https://github.com/Kurounin/PaginationExample) or [the react pagination example project](https://github.com/Kurounin/PaginationReactExample).

You can also checkout [this example application in React](https://github.com/mgscreativa/kurounin-pagination-react-example) created by [mgscreativa](https://github.com/mgscreativa).


# Server Pagination settings available on init

* `name`: set the publication name (defaults to **collection name**; *needs to be unique, to not collide with other publications*)
* `filters`: provide a set of filters on the server-side, which can not be overridden (defaults to **{}**, meaning no filters)
* `dynamic_filters`: provide a function which returns additional filters to be applied (**this** is the publication; receives no other parameters)
* `aggregate`: object describing the aggregation to be used on the publication
    * `pipeline`: the aggregation pipeline to execute
    * `observers`: an array of cursors. Each cursor is the result of a `Collection.find()`. Each of the supplied cursors will have an observer attached, so any change detected (based on the selection criteria in the `find`) will re-run the aggregation pipeline
    * `schema`: a SimplSchema instance required for ObjectIDs fields of aggregation result
* `transform_filters`: provide a function which returns the modified filters object to be applied (**this** is the publication; receives the current **filters** as an array containing the client & server defined filters and **options** as parameters)
* `transform_options`: provide a function which returns the modified options object to be applied (**this** is the publication; receives the current **filters** as an array containing the client & server defined filters and **options** as parameters)
* `countInterval`: set the interval in ms at which the subscription count is updated (defaults to **10000**, meaning every 10s)


# Client Pagination settings available on init

* `name`: set the subscription name (defaults to **collection name**; *needs to be identical with the server side publication name*)
* `page`: set the initial page, for example the page parameter from url (defaults to **1**)
* `perPage`: set the number of documents to be fetched per page (defaults to **10**)
* `skip`: set the number of documents that should be skipped when fetching a page (defaults to **0**)
* `filters`: filters to be applied to the subscription (defaults to **{}**, meaning no filters)
* `fields`: fields to be returned (defaults to **{}**, meaning all fields)
* `sort`: set the sorting for retrieved documents (defaults to **{_id: -1}**)
* `aggregateProps`: set the props to be used on aggregation pipeline (defaults to **{}**)
* `reactive`: set the subscription reactivity, allowing to only retrieve the initial results when set to false (defaults to **true**)
* `debug`: console logs the query and options used when performing the find (defaults to **false**)
* `connection`: the server connection that will manage this collection. Pass the return value of calling DDP.connect to specify a different server. (defaults to **Meteor.connection**)


# Client Pagination available methods

* `currentPage([int])`: get/set the current page
* `perPage([int])`: get/set the number of documents per page
* `skip([int])`: get/set the number of documents to skip
* `filters([Object])`: get/set the current filters
* `fields([Object])`: get/set the retrieved fields
* `sort([Object])`: get/set the sorting order
* `aggregateProps(Object)`: get/set the aggregate props
* `debug([boolean])`: get/set the debug
* `totalItems()`: get the total number of documents
* `totalPages()`: get the total number of pages
* `ready()`: checks if the subscription for the current page is ready
* `refresh()`: forcefully refreshes the subscription (useful for non-reactive subscriptions)
* `getPage()`: returns the documents for the current page


# Blaze Paginator template

A Blaze template is provided to allow navigation through available pages:

In the template html file add the paginator
```html
{{> defaultBootstrapPaginator pagination=templatePagination onClick=clickEvent limit=10 containerClass="text-center"}}
```
Available template parameters are:
* `pagination`: pagination instance
* `limit`: the maximum number of page links to display
* `containerClass`: optional container class for the paginator
* `paginationClass`: optional class for the *ul* element (defaults to `pagination`)
* `itemClass`: optional class for the page links elements
* `wrapLinks`: if set to true page links will be wrapped in *li* elements (defaults to `true`)
* `onClick`: optional callback to be called when page link is clicked (default callback runs `e.preventDefault()`)


# ReactJS Paginator class

A ReactJS class is provided to allow navigation through available pages:

```js
<DefaultBootstrapPaginator pagination={this.pagination} limit={10} containerClass="text-center" />
```
Available class properties are:
* `pagination`: pagination instance
* `limit`: maximum number of page links to display (defaults to **10**)
* `containerClass`: optional container class for the paginator

### Packages used as inspiration:

* [alethes:pages](https://atmospherejs.com/alethes/pages) for pagination instantiation
* [aida:pagination](https://atmospherejs.com/aida/pagination) for bootstrap paginator template
* [tunguska:reactive-aggregate](https://atmospherejs.com/tunguska/reactive-aggregate) Provides aggregation support to this library
