---
layout: default
title: Lazy Object Mapping with RestKit
---

In the Wanelo iOS application, we use RestKit to consume RESTful APIs. It's one of the few libraries that we've been
able to use without any modifications since the first version of Wanelo. In this post I'll try to explain how we used
Objective-C runtime to lazily map API responses into Obj-C objects using RestKit's object mapping.

<div style="float: right; margin: 20px">
    <img src="/assets/lazy_object_mapping_with_restkit/trending-products-sm.png" alt="Trending products grid" title="Trending products grid"/>
</div>

## The Problem

While developing our internal API for Wanelo's native iOS and Android applications, our approach was to provide not
only the data the apps need for their current state but also data for possible future states. For example, in the Trending
products view, only the product image is needed but the API also includes information about each product (e.g., price,
title) which is displayed only if the client navigates to the detail view of the product. The goal is to keep the app feeling fast,
and to reduce the number of HTTP calls made which have an overhead, especially on mobile networks.

However this also meant that the application was blocking the user while it was mapping information that
wouldn't be displayed immediately. For large API responses, this caused a significant delay. For
example, mapping the Trending products API response on an iPhone 4S was taking longer than 1 second.

## Solution

What we needed was a way to map only the immediately needed fields of an object, and map the rest of the fields when they are
accessed. Here's what our Trending products grid and product class looks like:



```objective-c
@interface WProduct : NSObject
@property(strong, nonatomic) NSURL *imageURL;
@property(strong, nonatomic) NSString *title;
@property(strong, nonatomic) NSNumber *price;
@property(strong, nonatomic) NSString *currency;
@end
```

It's clear that while the image for each product is being displayed, the other attributes (title, price and currency)
are not visible to the user. Therefore we only need to map the image property of the product object until the user navigates
to the product detail view where title, price and currency are displayed.

While doing this it would be ideal to have no changes to the interface of the WProduct class so that rest of the codebase stays the
same. With this requirement, the behavior of WProduct more or less matches a partially loaded object which is [often called a
"ghost" object](http://martinfowler.com/eaaCatalog/lazyLoad.html).

We also wanted to keep the option to fully map objects once the response is received, since we might have a different UI
where more properties are displayed. In this case we are lazily loading objects, because to load them fully after a short time
wouldn't have any benefits.

With these constraints we decided to implement lazy loading in the following way:

1. There should be a 'ghost' counterpart of each class which is the lazily loaded version so that the original class can be used
for the old behavior.

2. The 'ghost' version of each class should be substitutable for the actual class, so there is no need to change any existing code.
Subclassing the original class sounded like a good solution for this. So the ghost version of product is:

    ```objective-c
    @interface WGhostProduct : WProduct
    @end
    ```

3. Each lazily loaded class should have two lists of properties, one for initially loaded properties (like imageURL for
WProduct) and another for lazily loaded properties. We already had an internal protocol that all mapped classes need to
implement to provide information about how mapping should be done.

    ```objective-c
    @protocol WMappableResource <NSObject>
    @required
    //JSON keys => property names
    +(NSDictionary *)attributeMappings;
    @end
    ```

    This is how a simple WProduct can implement this protocol.

    ```objective-c
    @implementation WProduct
    + (NSDictionary *)attributeMappings {
        return @{
                @"image_url" : @"imageURL",
                @"title" : @"title",
                @"currency" : @"currency",
                @"price" : @"price",
                @"currency" : @"currency",
        };
    }
    ```

    If WGhostProduct implemented the same protocol with mappings for only initially loaded properties, then we can infer which properties are eagerly loaded and which are lazily loaded.

    ```objective-c
    @implementation WGhostProduct
    + (NSDictionary *)attributeMappings {
        return @{
                @"image_url" : @"imageURL",
        };
    }
    ```

4. Getter/Setter methods of lazily loaded properties would first fully map the object and then get/set the actual value.

    ```objective-c
    @implementation WGhostProduct
    ...
    - (NSString *)title {
        if(!self.fullyMapped) {
            [self mapFully];
        }
        return super.title;
    }

    - (void)setTitle:(NSString *)title {
        if(!self.fullyMapped) {
            [self mapFully];
        }
        [super setTitle:title];
    }
    @end
    ```

    The mapFully method can be implemented like this:

    ```objective-c
    - (void)mapFully {
        RKObjectMapping *mapping = [self mappingForWProductClass];
        RKMappingOperation *mappingOperation = [[RKMappingOperation alloc] initWithSourceObject:self.mappingSource
                                                                              destinationObject:self
                                                                                        mapping:mapping];
        [mappingOperation main];
        self.fullyMapped = YES;
    }
    ```

    Obviously this implementation requires a mappingSource, so we chose to store that in the ghost object by using the
mapperDidFinishMapping: mappingOperation:didSetValue:forKeyPath:usingMapping: methods of RKMapperOperationDelegate

    ```objective-c
    - (void)mapperDidFinishMapping:(RKMapperOperation *)mapper {
        if ([mapper.mappingResult.firstObject isKindOfClass:[WGhostProduct class]]) {
            WGhostProduct *ghostProduct = mapper.mappingResult.firstObject;
            ghostProduct.mappingSource = mapper.representation;
        }
    }
    - (void)mappingOperation:(RKMappingOperation *)operation
                 didSetValue:(id)value
                  forKeyPath:(NSString *)keyPath
                usingMapping:(RKPropertyMapping *)propertyMapping {
        if ([value isKindOfClass:[NSArray class]]) {
            NSArray *mappedValueArray = value;
            NSArray *sourceValues = (operation.sourceObject)[propertyMapping.sourceKeyPath];
            for (NSUInteger index = 0; index < mappedValueArray.count; index++) {
                id mappedValue =  mappedValueArray[index];
                if ([mappedValue isKindOfClass:[WGhostProduct class]]] && sourceValues.count > index) {
                    WGhostProduct *ghostProduct = mappedValueArray[index];
                    ghostProduct.mappingSource = sourceValues[index];
                }
            }
        } else if ([value isKindOfClass:[WGhostProduct class]]) {
            WGhostProduct *ghostProduct = value;
            id sourceValue = operation.sourceObject;
            if (propertyMapping.sourceKeyPath.length > 0) {
                sourceValue = sourceValue[propertyMapping.sourceKeyPath];
            }
            ghostProduct.mappingSource = sourceValue;
        }
    }
    ```

This summarizes our approach to implementing lazy mapping with RestKit. However the code examples I provided can be improved
upon quite a bit by using Obj-C runtime and protocols instead of concrete classes:

* Overriding all the getter/setter methods in each ghost class is tedious and error prone. We dynamically added getter/setter
methods that also fully maps the object using class_addMethod(), and then swapped them with regular getter/setter methods
using method_exchangeImplementations().

* We defined a WLiteMappable protocol that all ghost classes implement, so that while setting mappingResource we don't
need to work with concrete classes like WGhostProduct

Overall this solution helped us reduce the mapping time for the Trending products API response from over 1 second to under 250 milliseconds
with a very small number of changes in the rest of the codebase. But it's not without its shortcomings.

## Caveats

* Mapping of lazily mapped properties is done on the same thread that getter/setter methods are called. For most cases
this will be the main thread. If mapping takes a long time, it will degrade the user's experience.

* Another problem with threading is the non-thread-safe classes (like NSDateFormatters) that RKObjectMapping uses. We had to insert
new instances of date formatters to mapping objects to avoid this problem.

* Adding and swizzling methods in runtime can lead to inscrutable stack traces and painful debugging.

* Adding and swizzling methods take time, and it can slow down the application launch time if it is done immediately after the application
starts.

* If isEqual: and hash methods use lazily mapped properties, using ghost objects in collections may cause many of them to be
fully mapped, resulting in long mapping calls in the main thread.

* Similarly, if the UI starts to show a property that is lazily loaded, it is easy to forget to adjust the ghost class to move that
field to eagerly mapped list. This can lead to fully mapping many objects in the main thread.

* We don't know if this approach would work if Core Data is used with RestKit.

-- [Server](http://wanelo.com/server "Server on Wanelo")
