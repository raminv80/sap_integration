[logo]: https://avatars3.githubusercontent.com/u/1459952?v=3&s=200 "Logo Title Text 2"
[diagram]: ./components.png "SAP Integration software components diagram"

SAP Integration Module
========================

This module is designed to watch over changes on Magento's internal models and exports selected models to target systems.


## Setup Instructions:
 ### 1. Setup a Queue Server

This module uses magento-lilmuckers_queue (https://github.com/lilmuckers/magento-lilmuckers_queue) module to process jobs in a queue. Define your queue server in app/etc/local.xml
For example to use gearman as the queue processor add following configuration to your local.xml file.

```xml
<config>
    <global>
        <queue>
            <gearman>
                <servers>
                    <server1>
                        <host>127.0.0.1</host>
                        <port>4730</port>
                    </server1>
                </servers>
            </gearman>
            <backend>gearman</backend>
        </queue>
    <global>
<config>
```

### 2. Setup Target Integration

Let's assume we are planning to integrate Magento to staging database of target system named 'SAP Integration'.
All configuration of this integration are placed under namespace of global/sapintgeration. Here you can disable/enable the integration per target and define a separate processing queue.

```xml
<config>
    <global>
        <sapintgration>
            <active>1</active>
            <queue>sap_integration_queue</queue>
        </sapintgeation>
    </global>
</config>
```

### 3. Tell module how to map data.
To Do so you need to tell the module:

- The name of model in target system

  list the names of target models under map entry and keep the name less than 50 characters:

```xml
<config>
    <global>
        <sapintgration>
            <active>1</active>
            <queue>sap_integration_queue</queue>
            <map>
                <nameOfTargetModel1></nameOfTargetModel1>
                <nameOfTargetModel2></nameOfTargetModel2>
            </map>
        </sapintgeation>
    </global>
</config>
```

- The class of tracked Magento model

  Each target model should *watch* over changes on a Magento model. Take note that it expects the name of the class rather than its alias.

```xml
<config>
    <global>
        <sapintgration>
            <active>1</active>
            <queue>sap_integration_queue</queue>
            <map>
                <nameOfTargetModel1>
                    <watch>MagentoModelClass1</watch>
                </nameOfTargetModel1>
                <nameOfTargetModel2>
                    <watch>MagentoModelClass</watch>
                </nameOfTargetModel2>
            </map>
        </sapintgeation>
    </global>
</config>
```

 - Field to field mapping instruction

   Each model's fields in target system can be mapped to a method or a data defined on:
    * The Magento's model being watched over.
    * Its **mapping** class.

   Let's assume there is an Order model in target system expecting order number as OrderNum, customer name as cardName and shipping country as ShipToCountry. The integration needs to watch over magento's order model (Mage_Sales_Model_Order) and map fields as expected.
   order number is same as entity_id which is available on Mage_Sales_Model_Order instance data as entity_id and is being used as primary key in target model. The Mage_Sales_Model_Order instance provides a method to get order's customer's name by calling getCustomerName method. Finally, To get shipping country, there are no direct methods defined on the Mage order instance, as such we need to define an arbitrary getter in mapper class to construct the information we need (we will get back on mapper class in  a bit). Configuration of this mapping is as follow:

```xml
<config>
    <global>
        <sapintgration>
            <active>1</active>
            <queue>sap_integration_queue</queue>
            <map>
                <order>
                    <watch>Mage_Sales_Model_Order</watch>
                    <map>
                        <orderNum index="primary">entity_id</orderNum>
                        <cardName>getCustomerName</cardName>
                        <shipToCountry>getShipToCountry</shipToCountry>
                    </map>
                </order>
            </map>
        </sapintgeation>
    </global>
</config>
```
If you need to write changes of two different models to one target model then you need to explicitly `rename` target models. For example to map both 'Mage_Sales_Model_Order' and 'Mage_Sales_Model_Order_dummy' to a single target model named 'order' do:
```xml
<config>
    <global>
        <sapintgration>
            <active>1</active>
            <queue>sap_integration_queue</queue>
            <map>
                <order>
                    <watch>Mage_Sales_Model_Order</watch>
                    <map>...</map>
                </order>
                <order_dummy>
                    <watch>Mage_Sales_Model_Order_dummy</watch>
                    <map>...</map>
                    <rename>order</rename>
                </order_dummy>
            </map>
        </sapintgeation>
    </global>
</config>
```

This powerful method gives you the ability to merge multiple magento models into one target model. A scenario is exporting shipping as an order item to the target system.

Finally define `indexes`. An index should make target record unique. The module uses index to performs **updates**. If no indexes are defined on a model then all exported jobs will be **Inserted** to target system. With index in place, if there is a matching record then the record will be *Updated*. 

There are two different types of indexes: 

##### 1. Simple Primary Index
Defines a subset of target model's fields as primary keys of the model. In the following example, field *orderNumber* of *order*'s target model is set to be a simple primary. This means for every export job, module first looks for a record in target system using *orderNum* and updates the first record found or inserts a new one.
```xml
<config>
    <global>
        <sapintgration>
            <active>1</active>
            <queue>sap_integration_queue</queue>
            <map>
                <order>
                    <watch>Mage_Sales_Model_Order</watch>
                    <map>
                        <orderNum index="primary">entity_id</orderNum>
                        <cardName>getCustomerName</cardName>
                        <shipToCountry>getShipToCountry</shipToCountry>
                    </map>
                </order>
            </map>
        </sapintgeation>
    </global>
</config>
```   

Note that you can have multiple simple primary keys on multiple fields of a model.

##### 2. Mapped indexes 
If there is no direct relation between indexes of target model and values being written to target system (for example the ID in target system is set to be auto increment) mapped indexes provide a mechanism to map internal index to stored index.

Assume payments are to be exported to a database with *paymentnum* field set to be auto increment and a field named *amount*. In this example, what makes source model unique is the entity_id on payments model and what makes target's model unique is the autoIncrement Id of payment record. The mapping config looks something like:
```xml
<payments>
<watch>Mage_Sales_Model_Order_Payment</watch>
<map>
    <payment_num type="dummy" index="primary">autoIncrement</payment_num>
    <paymentEntityId type="dummy" index="mapped">entity_id</paymentEntityId>
    <paymentTotal>getPaymentTotal</paymentTotal>
    <paymentType>getPaymentType</paymentType>
</map>
</payments>
```

Fist, target field *payment_num* defines the primary key on target's model using attribute `index="primary"`. The a set of fields are being mapped as keys making the record unique on source model using attribute `index="mapped"`. Take note that in this example none of paymentNum or PaymentEntityId should be written to target system. To achieve this effect, both fields are marked as dummy using attribute `type="dummy"`. You may also noticed the auxiliary autoIncrement method defined for payment_num. This method is just for notion and will not generate any output. 

What happens in backend is that for every job module collects information of fields tagged as mapped and looks for their corresponding record in *sapintegration_map_index* and  *sapintegration_map_index_key* tables. If non found system simply *inserts* the data to target system and saves auto value of field tagged as primary in *sapintegration_map_index* table and all the mapped fields for this record in *sapintegration_map_index_key* table. If corresponding record was found it extracts the mapped index from *sapintegration_map_index* table and *updates* the record.

#### Mapping classes
Finally for every target model listed under map you **must** create a mapping class. All arbitrary getters for your mapping can be defined in these mapping classes. Create **local/SapIntegration/Model/Stagingdb/Map/Order.php** with following content:

```php
<?php
class Aligent_SapIntegration_Model_Stagingdb_Map_Order extends Aligent_SapIntegration_Model_Map_Abstract{

    /**
     * @param Mage_Sales_Model_Order $order
     * @return string
     */
    public function getShipToCountry(Mage_Sales_Model_Order $order){
        return $order->getShippingAddress()->getCountry();
    }
```

The validate method is used for conditional export. If validate function returns false, module will skip exporting changes for that model.For example to only export *closed* orders the mapper class can add a validate method as follow:

```php
<?php
class Aligent_SapIntegration_Model_Stagingdb_Map_Order extends Aligent_SapIntegration_Model_Map_Abstract{

    public function validate(Mage_Sales_Model_Order $order){
        return $order->getStatus() == 'closed';
    }
    
    ...
}
```

### 4. Delegated Methods
Delegation is a mechanism where generation of target value is delayed till after queue processing within the background processing job. Delegated methods are useful to push heavy calculations to the backend as well as giving the opportunity to resolve object dependencies. Consider following example:

An integration exports order *contact info* and *order info* as *contact* and *order* objects where *order* has a field pointing to exported *contact* ID. At time of job creation, *contact* is not yet exported to target system and there is no way to obtain its target ID. By delegating calculation of order>contact_ID  to the background process we can make sure its value will be assigned after writing *contact* details to target system.

Configuration of delegation is done using:
1. Declaring `delegate` attribute on Target Field.
2. Assigning callback function as the value of delegate attribute.
3. Add parameters for callback function as children of target field.
4. Define callback function on the mapper model. All parameters will be send to the callback function as an array to the first parameter of function.

Here is implemented sample of delegated order contact info:

```xml
<order>
    <watch>Mage_Sales_Model_Order</watch>
    <map>
        ...
        <ContactId delegate="getDelegatedBillingContactId">
            <magentoContactId>getOrderContactId</magentoContactId>
            <websiteId>getWebStoreId</websiteId>
        </ContactId>
        ...
```
```php
class Aligent_SapIntegration_Model_Target_Map_Order extends Aligent_SapIntegration_Model_Map_Abstract{
    ...
    public function getDelegatedBillingContactId(Array $obj){
        if($obj['magentoContactId']){
            return $this->getTargetContactId($obj['magentoContactId'], $obj['websiteId']);
        }
    }
    
    public function getOrderContactId(Mage_Sales_Model_Order $obj){
        return ...
    }
    
    public function getWebStoreId(Mage_Sales_Model_Order $obj){
    ...
```

### 5. Define an **adapter** responsible to write mapped data to target system
An adapter class should extend `Aligent_SapIntegration_Model_Adapter_Abstract` and provide a  public `export` method. This method accepts three parameters: 
- $action: which is type of action (insert/update/delete)
- $model: the name of target model
- $dataLoad: mapped model data

Foir example:
```php
class Aligent_SapIntegration_Model_Stagingdb_Adapter extends Aligent_SapIntegration_Model_Adapter_Abstract
{
    protected $_connection = false;
    public function _construct(){
        $this->_connection = Mage::getSingleton('core/resource')->getConnection('sap_staging_db');
    }

    public function export($action, $model, $dataLoad){
        //export $dataLoad as model named $model
    }
}
```
---
## Application Architecture

![Application architecture diagram][diagram]

Developer
---------
Ramin Vakilian <ramin@aligent.com.au>

Licence
-------
[OSL - Open Software Licence 3.0](http://opensource.org/licenses/osl-3.0.php)

Copyright
---------
(c) 2015 Aligent Consulting

![Aligent Consulting Group][logo]