#### 3.9.3.1 Installation and Update
If a Key-Value Storage was not opened before, an `installation` is conducted and `the configured initial key-value pairs` are installed into the location of the Key-Value Storage.（因为kvs可以配置到文件系统的任意位置，所以安装就是把etc/xxx.json拷贝到设置的位置） After the installation the Key-Value Storage is registered by Persistency.
The next time Persistency is initialized and the same Key-Value Storage is opened, there will be an attempt to update this Key-Value Storage.
**The update is triggered if `the software cluster version` or `the executable version` of the current configuration is newer than in the configuration of the run before.**
Moreover, in case the executable version was updated the data of the Key-Value Storage can be modified by the application using the `RegisterApplicationDataUpdateCallback`.

#### 3.9.3.2 Key-Value Storages in safety environments
If a Key-Value Storages is used in a safety environment, the target identifier of the integrity file stream is required to be unique. Therefore, the Key-Value Storage file path must be `an absolute path` that is not allowed to be changed. Moreover, `safe Key-Value Storages` need to have a model extension configured in the AUTOSAR model to indicate that they are used in a safety environment. More information about this model extension can be found in chapter 5.2.5.
In case a Key-Value Storage is used in a safety environment and the data is read from a non-safe Key-Value Storage data file, the user is responsible to validate and migrate the restored data.

#### 3.9.3.3 Key-Value Storage file format
The data of a Key-Value Storage are stored with a proprietary format in a file. The Key-Value Storage file is not meant for manual editing.

### 3.9.4 Update of Persistent Data
#### 3.9.4.1 Allowed Modifications in ARXML models
The possible modifications for an updated ARXML model are summarized in Table 3-4 and Table 3-5. It is entirely possible to modify every parameter in an updated ARXML model - even if a parameter is listed as non-modifiable. In this case the updated set of parameters can lead to a storage that cannot be opened anymore (e.g. if the redundancy settings are modified to a more stringent setup). In this situation the affected storage can either be deleted or reset using the respective public API. In both cases data is lost.

表 3-4 和表 3-5 总结了对更新的 ARXML 模型可能进行的修改。完全可以修改更新的 ARXML 模型中的每个参数，即使某个参数被列为不可修改。在这种情况下，更新后的参数集可能导致存储无法再打开（例如，如果冗余设置被修改为更严格的设置）。在这种情况下，可以删除受影响的存储或使用相应的公共 API 重置存储。在这两种情况下，数据都会丢失。

#### 3.9.4.2 Migration Steps to update a Key-Value Storage Data Type
In order to update a Key-Value Storage data type `there are modifications necessary in the ARXML model and in the implementation`. The latter migrates the data during runtime.
Please Note:
> The old data type and its name must not be modified! Instead, **a new data type has to be created and the data has to be migrated**.
These steps are necessary in order to update the ARXML model:
> Do NOT delete the old data type!
> If not already done, publish the old data type as a `dataTypeForSerialization` in the respective Key-Value Storage interface.
> For all initial values of the old data type set the update strategy to `KEEP-EXISTING`. If the old values are not needed anymore, the update strategy could also be DELETE.
> Create the new data type (which is based on the old data type).
> If initial values for the new data type are needed, create them with the update strategy `OVERWRITE or KEEP-EXISTING`.
The data migration takes place during runtime in the application data updater. Therefore, the user has to register an appropriate application data updater (using `ara::per::RegisterApplicationDataUpdateCallback`).
The application data updater is invoked in the call to `ara::per::UpdatePersistency()` or `ara::per::OpenKeyValueStorage()`.
As configured, the data of the old type is not touched and the data of the new type is installed. Following steps are necessary to migrate the data of the old type:
> Load the data of the old type using `ara::per::KeyValueStorage::GetValue()`.
> Load the data of the new type using `ara::per::KeyValueStorage::GetValue()`.
> Migrate the relevant data elements from the old type to the new type.
> Overwrite the data of the new type using `ara::per::KeyValueStorage::SetValue()`.
If the key of the new type data shall be equal to the key of the old type data, the Key-Value pair of the old type data must be deleted (using `ara::per::KeyValueStorage::RemoveKey()`) prior to storing the new type data with the original key.



### 5.2.3 Configuration of Initial Values for Persistent Data
Initial values for persistent data can be configured during both `design phase` and `deployment phase`.

Configuration of `initial values` for a PersistencyKeyValueStorageInterface in the `design phase` is done via a PortPrototype typed by the `PersistencyKeyValueStorageInterface`. On such a PortPrototype, initial values are represented by the meta-class `PersistencyDataRequiredComSpec`, which aggregates a reference to the data element in the PersistencyKeyValueStorageInterface to which it applies, and a ValueSpecification, providing the initial value for the data element.
在设计阶段配置 PersistencyKeyValueStorageInterface 的初始值是通过 PersistencyKeyValueStorageInterface 类型的 PortPrototype 来完成的。在这样的 PortPrototype 上，初始值由元类 PersistencyDataRequiredComSpec 表示，元类 PersistencyDataRequiredComSpec 集合了对 PersistencyKeyValueStorageInterface 中数据元素的引用（该引用适用于 PersistencyKeyValueStorageInterface），以及提供数据元素初始值的 ValueSpecification。

In the deployment phase, initial values for a PersistencyKeyValueStorage can be directly configured via `PersistencyKeyValuePairs`. The meta-class PersistencyKeyValuePair aggregates a ValueSpecification, representing the initial value for the PersistencyKeyValuePair, and a reference to an `AbstractImplementationDataType`.
在部署阶段，可通过 PersistencyKeyValuePair 直接配置 PersistencyKeyValueStorage 的初始值。元类 PersistencyKeyValuePair 集合了代表 PersistencyKeyValuePair 初始值的 ValueSpecification 和对 AbstractImplementationDataType 的引用。

For File Storage, the configuration of initial file content in the design phase is done via a PersistencyFileStorageInterface that aggregates PersistencyFileElements; the PersistencyFileElements in turn aggregate a UriString that specifies a URI to the initial contents for the PersistencyFileElements. In the deployment phase, the configuration of initial file content can be achieved by the meta-class PersistencyFile. A PersistencyFileStorage aggregates PersistencyFiles, which in turn aggregate UriStrings that specify a URI to the initial contents for the PersistencyFiles.
对于文件存储，设计阶段的初始文件内容配置是通过 PersistencyFileStorageInterface 完成的，该接口聚合了 PersistencyFileElements；PersistencyFileElements 又聚合了一个 UriString，该 UriString 指定了 PersistencyFileElements 初始内容的 URI。在部署阶段，初始文件内容的配置可通过元类 PersistencyFile 来实现。PersistencyFileStorage 集合了 PersistencyFiles，而 PersistencyFileStorage 又集合了 UriStrings，后者指定了 PersistencyFiles 初始内容的 URI。

Note that the configuration from the deployment always overrules the configuration from the design, as described in section 5.2.1.
For signed and unsigned integer types ( int8 , ..., int64 , uint8 , ..., uint64 ) and the bool data type, as well as for user-defined enumeration and bitfield data types, it is possible to specify initial values via a NumericalValueSpecification by providing a decimal, hexadecimal, octal or binary literal. For bool, also the string literals true and false (in any capitalization) are accepted. As an example, specifiying a uint8 initial value of 32 is possible via the literals 32 (decimal), 0x20 (hexadecimal), 0o40 (octal) and 0b100000 (binary). The specification of negative values for signed types via hexadecimal, octal and binary literals is also possible by prepending a minus sign to the (positive) literal: the hexadecimal
literal -0x20 specifies the decimal value -32 .
请注意，如第 5.2.1 节所述，**部署中的配置总是优先于设计中的配置**。
对于有符号和无符号整数类型（int8 , ..., int64 , uint8 , ..., uint64 ）和 bool 数据类型，以及用户定义的枚举和位域数据类型，可以通过 `NumericalValueSpecification` 提供十进制、十六进制、八进制或二进制字面来指定初始值。对于 bool，也接受字符串字面 true 和 false（任意大小写）。例如，可以通过字面 32（十进制）、0x20（十六进制）、0o40（八进制）和 0b100000（二进制）来指定 uint8 的初始值为 32。通过十六进制、八进制和二进制字面也可以指定有符号类型的负值，方法是在（正）字面前加上一个减号：十六进制字面 -0x20 指定十进制值 -32。

For user-defined bitfield types, it is also possible to specify initial values via a TextValueSpecification. The initial value can be specified by providing a string literal, containing the named entries (bits) of the associated CompuMethod of the bitfield that should be set in the initial value. Multiple named entries have to be separated by the ’|’ character, which represents the bitwise or operation. As an example, if the associated CompuMethod of a bitfield type has the named entries front_left with value 0b01 and front_right with value 0b10 for the lowest bit and second lowest bit, respectively, providing the initial value as "front_left|front_right" will set the initial value of the bitfield to 0b11 = 3 .
For user-defined variant types, it is possible to specify initial values via a RecordValueSpecification. The initial value of a variant must always contain two fields. The first field must be a NumericalValueSpecification containing the zero-based index of the alternative and the second field must contain the actual value.
For details about the configuration of bitfield values, consult the AUTOSAR references [1] and [2].

对于用户定义的位字段类型，也可以通过 TextValueSpecification 指定初始值。
初始值可以通过提供一个字符串来指定，该字符串包含应在初始值中设置的位字段相关 CompuMethod 的命名项（位）。
多个命名项必须用“|”字符分隔，“|”表示位操作或操作。举例来说，如果位字段类型的相关 CompuMethod 有命名项 front_left（值为 0b01）和 front_right（值为 0b10），分别代表最低位和第二低位，那么以 “front_left|front_right ”的形式提供初始值将把位字段的初始值设置为 0b11 = 3。
对于用户定义的变量类型，可以通过 RecordValueSpecification 指定初始值。变量的初始值必须始终包含两个字段。第一个字段必须是一个 `NumericalValueSpecification`，包含备选值的零基索引，第二个字段必须包含实际值。有关位值配置的详细信息，请参阅 AUTOSAR 参考文献 [1] 和 [2]。




























