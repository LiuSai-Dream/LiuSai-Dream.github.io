## Tensorflow版本语义
### 语义版本控制
Tensorflow使用语义版本2.0，格式如下`MAJOR.MINOR.PATCH` ,每个数字的意义如下：
- **MAJOR**：向后版本兼容。前一个版本的代码和数据未必和新版本的一致。然后，一些已有的数据（图，检查点，其他protobufs）可能会具有移植性。
- **MINOR**：后向兼容特征、速度等。
- **PATCH**：后向bug修复
