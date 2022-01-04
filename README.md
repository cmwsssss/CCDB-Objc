中文版本请[点击这里](https://github.com/cmwsssss/CCDB-OBJC/blob/main/README-CN.md)

# CCDB-OBJC
CCDB-OBJC is a high-performance database framework based on Sqlite3 and OBJC written to enable users to perform the simplest and fastest data processing operations at the application level.

CCDB has a Swift version, the Swift version of CCDB adapted SwiftUI, using SwiftUI or use pure Swift developers please [click here to view](https://github.com/cmwsssss/CCDB)

## Features

### Easy-to-use:
CCDB-OBJC tries to provide the easiest and fastest way to use，without any additional configuration code to achieve **dictionary -> model <-> database** data mapping, only one line of code is needed to insert, update, delete, and query, and the programmer does not need to pay attention to any database level operations, such as transactions, database connection pooling, thread safety, etc. CCDB will optimize the API operations at the application level to ensure efficient operation at the database level.

### Efficient:
CCDB-OBJC is based on the multi-threaded model of sqlite3 to work, and it has memory caching mechanism, making its performance better than native sqlite3 most of the time, and the OC version of CCDB uses lazy loading for complex properties, so it performs very well on complex models.

### Container:
CCDB-OBJC also provides a list solution: Container, which makes it very easy to save and read list data.

### Dictionary data mapping:
CCDB-OBJC can map dictionary data to the model without any additional mapper code, making your data processing work much easier.

#### Singleton:
The object generated by CCDB-OBJC will only have one copy in memory

## Getting Started

### Prerequisites
Apps using WCDB can target: iOS 6 or later.


### Installation
pod 'CCDB-OBJC'

### Initialize
Call the initialization method before using the CCDB's API
```
[CCDB initializeDBWithVersion:@"1.0"];
```
If the data model properties have changed and you need to migrate the database, just change the version

### Object Relational Mapping(ORM)

#### Inheritance

The model to access CCDB needs to inherit from CCModel.

```
@interface UserModel : CCModel
...
@end
```
#### Property：
The default types supported by CCDB are: int, double, float, bool, NSString and classes that implement the CCDBSaving protocol

CCDB provides 4 types of property declarations, and CCDB only binds the properties declared by these macro to the data table。

##### 1. CC_PROPERTY
The usual way of declaring properties for the types supported by CCDB
```
/*
CC_PROPERTY(policy, classType, propertyName)
policy: The policy of the property, e.g. noatomic
classType: The type of the property, e.g. NSString *
propertyName: The property name of the property
*/

@interface UserModel : CCModel

CC_PROPERTY((nonatomic, assign), NSInteger, userId);
CC_PROPERTY((nonatomic, strong), NSString *, username);

@end


```

##### 2. CC_PROPERTY_JSON
If you want CCDB to automatically map data from dictionary data to the model, you need to use CC_PROPERTY_JSON to declare properties
```
/*
CC_PROPERTY_JSON(policy, classType, propertyName, keyPath)
keyPath Maps the value specified in the dictionary to the model via keyPath, which is separated by a double underscore (__) to separate the hierarchy

For example, if you need to map the age of the following data into the model, the keyPath is info__age
{
    "userId" : 1
    "username" : "Francis"
    "info" : {
        "age" : 30
    }
}

*/

@interface UserModel : CCModel

CC_PROPERTY_JSON((nonatomic, assign), NSInteger, userId, userId);
CC_PROPERTY_JSON((nonatomic, strong), NSString *, username, username);

//keyPath is info__age
CC_PROPERTY_JSON((nonatomic, assign), NSInteger, age, info__age);
@end

*/
```

##### 3. CC_PROPERTY_TYPE
If you wish to bind a type not supported by CCDB to database table, you need to use CC_PROPERTY_TYPE for the property declaration

The enum of property types for CCDB:

```
typedef NS_ENUM(NSUInteger, CCModelPropertyType) {
    //default
    CCModelPropertyTypeDefault = 1,
    
    //The declared property is a subclass of CCModel
    CCModelPropertyTypeModel,
    
    //the declared property is a serializable dictionary or array
    CCModelPropertyTypeJSON,
    
    //the declared property is a custom type
    CCModelPropertyTypeCustom,
    
    //the declared property follows the CCDBSaving protocol
    CCModelPropertyTypeSavingProtocol,
};
```

###### 1. CCModelPropertyTypeModel
If the property type is a subclass of CCModel, the property needs to be declared with this type, and CCDB will write to the subproperty together with the write operation when writing, and also when querying
```
@interface MessageModel : CCModel

CC_PROPERTY((nonatomic, strong), NSString *, messageId);
CC_PROPERTY_TYPE((nonatomic, strong), UserModel *, user, CCModelPropertyTypeModel);

@end
```

###### 2. CCModelPropertyTypeJSON
If the property type is serializable NSArray and NSDictionary, CCDB will automatically serialize it as a string and write into DB, and automatically map it to NSArray and NSDictionary data when reading from DB

**CCDB does not support query on the key of dictionary for now**
```
@interface MomentModel : CCModel

CC_PROPERTY((nonatomic, strong), NSString *, momentId);
CC_PROPERTY_TYPE((nonatomic, strong), NSArray *, comment, CCModelPropertyTypeJSON);

@end
```

###### 3. CCModelPropertyTypeCustom
If the property is a custom property, such as a subClass of NSObject and does not follow the CCDBSaving protocol, you need to use CCModelPropertyTypeCustom for property declaration and implement the encoding and decoding methods

```
@interface MomentModel : CCModel

CC_PROPERTY((nonatomic, strong), NSString *, momentId);
CC_PROPERTY_TYPE((nonatomic, strong), MyModel *, myModel, CCModelPropertyTypeCustom);

@end

@implementation MomentModel
 
 
 - (NSMutableDictionary *)customJSONDictionary {
     NSMutableDictionary *dic = [[NSMutableDictionary alloc] init];
     //Encode the data as a dictionary
     [dic setObject:myModel.foo forKey:@"myModel_foo"];
     return dic;
 }

 - (void)updateDataWithCustomJSONDictionary:(NSMutableDictionary *)dic {
     //Decode data from the dictionary
     self.myModel = [[MyModel alloc] init];
     myModel.foo = [dic objectForKey:@"myModel_foo"];
 }
 
 @end
 
```

###### 4. CCModelPropertyTypeSavingProtocol
For classes that follow the CCDBSaving protocol, you need to use the CCModelPropertyTypeSavingProtocol to declare properties

**Classes that follow CCDBSaving do not need a primary key, their properties are declared in the same way as CCModel, their objects cannot be written to the database alone, they must exist as properties of CCModel**

```
@interface MyObject : NSObject <CCDBSaving>

CC_PROPERTY((nonatomic, strong), NSString *, foo_1)
CC_PROPERTY((nonatomic, strong), NSString *, foo_2)

@end
 
@interface MomentModel : CCModel

CC_PROPERTY((nonatomic, strong), NSString *, momentId);
CC_PROPERTY_TYPE((nonatomic, strong), MyObject *, myObject, CCModelPropertyTypeSavingProtocol);

@end
```

CCDB will automatically serialize and deserialize the properties of MyObject, if you need to customize the serialization process, you need to implement the following methods

**If you implement the following method, the CCDB automatic mechanism will fail, please make sure all properties are handled properly**
```
 - (void)cc_updateWithJSONDictionary:(NSDictionary *)dic {
    //swap foo_1 and foo_2 data when reading
    self.foo_1 = [dic objectForKey:@"foo_2"];
    self.foo_2 = [dic objectForKey:@"foo_1"];
 }
 
 - (NSMutableDictionary *)cc_JSONDictionary {
    NSMutableDictionary *dic = [[NSMutableDictionary alloc] init];
    //encode the data as a dictionary
    [dic setObject:self.foo_1 forKey:@"foo_1"];
    [dic setObject:self.foo_2 forKey:@"foo_2"];
    return dic;
 }
```
##### 4. CC_PROPERTY_TYPE_JSON
If you want CCDB to map dictionary data to custom type properies, you need to declare properties using this marco. The keyPath mechanism is described above, so we won't repeat it here.
```
@interface MessageModel : CCModel

CC_PROPERTY((nonatomic, strong), NSString *, messageId);
CC_PROPERTY_TYPE_JSON((nonatomic, strong), UserModel *, user, CCModelPropertyTypeModel, user);

@end
```

### Declare a primary property
CCDB requires that each data table must have a primary proeprty, you need to declare the primary property in the .m file

```
@implementation UserModel

//userId is the property name of the primary property
CC_MODEL_PRIMARY_PROPERTY(userId)

@end
```

### Update and Insert
For CCDB, the operations are based on CCModelSavingable objects, **objects must have primary property**, so update and insert are the following code, if there is no data corresponding to that primary key within the data, it will be inserted, otherwise it will be updated. 

**CCDB does not provide batch write, CCDB will automatically create write transactions and optimize them.**
```
[userModel replaceIntoDB];
```

### Query
CCDB support query by primary key, batch queries and conditional queries

##### Query by primary key
Get the corresponding model object by primary key
```
UserModel user = [[UserModel alloc] initWithPrimaryProperty:@"userId"];
```
#### Batch queries
* Get the count of the table
```
NSInteger count = [UserModel count];
```
* Get all objects of the model
```
NSArray *users = [UserModel loadAllDataWithAsc:false];
```

##### Conditional queries
The configuration of the CCDB condition is done through the object of the CCModelCondition
Query the first 30 users whose age is greater than 20 in the UserModel table, and return the results in reverse order by age
```
CCModelCondition *condition = [[CCModelCondition alloc] init];

//ccmethods are not sequential
condition.ccWhere(@"Age > 30").ccOrderBy("Age").ccLimited(20).ccOffset(0).ccIsAsc(false);

//Query users according to the conditions
NSArray *res = [UserModel loadDataWithCondition:condition];

//Get count of users according to the conditions
NSInteger count = [UserModel countBy:condition];
```

### Dictionary mapping to model
CCDB can automatically map the dictionary to the model based on the configuration of the property declaration by calling the following method
```
UserModel *user = [[UserModel alloc] initWithJSONDictionary:dic];
```

### Delete
* Delete an object
```
[userModel removeFromDB];
```
* Delete all objects
```
[UserModel removeAll];
```

#### Index
* Create index
```
//Create index for age
[UserModel createIndexForProperty:@"Age"];
```
* Remove index
```
//Remove index for age
[UserModel removeIndexForProperty:@"Age"];
```

#### Container
Container is a solution for list data, the value of each list can be written to Container, Container's table data is not a separate copy, its associated with the data table data

```
Car *glc = [[Car alloc] init];
glc.name = @"GLC 300"
glc.brand = @"Benz"

//Assuming the containerId of the Benz car is 1, here the glc will be written into the list container of the Benz car
[glc replaceIntoDBWithContainerId:1 top:false];

//Remove glc from the list of Benz cars
[glc removeFromContainer:1];

//Get the list data of all Benz cars
NSArray *benzCars = [Car loadAllDataWithAsc:false containerId:1];
```
Container data access has also been optimized within CCDB

