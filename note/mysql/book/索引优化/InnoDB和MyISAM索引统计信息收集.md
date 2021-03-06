## 5.6 InnoDB和MyISAM索引统计信息收集

存储引擎收集有关表的统计信息，以供优化器使用。表统计信息基于值组，其中值组是一组具有相同键前缀值的行。出于优化目的，重要的统计数据是平均值组的大小。

MySQL通过以下方式使用平均值组大小：

- 估算每次`ref`访问 必须读取多少行

- 估计部分连接将产生多少行；也就是说，这种形式的操作将产生的行数：

  ```sql
  (...) JOIN tbl_name ON tbl_name.key = expr
  ```

随着索引的平均值组大小的增加，索引对于这两个目的的用处不大，因为每次查找的平均行数增加：为了使索引更好地用于优化目的，最好将每个索引值作为目标表中的行数。当给定的索引值产生大量的行时，该索引的作用较小，而MySQL不太可能使用该索引。

平均值组的大小与表基数有关，表基数是值组的数目。该 `SHOW INDEX`]语句显示基于的基数值*`N/S`*，其中 *`N`*是表中的行数，并且*`S`*是平均值组的大小。该比率在表中产生大约数量的值组。

对于基于联接`<=>`比较运营商，`NULL`没有从任何其它值区别对待：`NULL <=> NULL`，就像任何其他 。 `*`N`* <=> *`N`*`*`N`*

但是，对于基于`=`运算符的联接， `NULL`它与非`NULL`值是不同的： 当或 （或两者）均为 时不为真 。这会影响 以下形式的比较访问：如果的当前值是，MySQL将不会访问表 ，因为比较不能为真。 `*`expr1`* = *`expr2`*`*`expr1`**`expr2`*`NULL``ref``*`tbl_name.key`* = *`expr`*`*`expr`*`NULL`

为了`=`进行比较，`NULL`表中有多少个值都没有关系。为了优化目的，相关值是非`NULL`值组的平均大小。但是，MySQL当前不支持收集或使用该平均大小。

对于`InnoDB`和`MyISAM` 表，您分别可以通过`innodb_stats_method`]和 `myisam_stats_method`系统变量来控制对表统计信息的收集 。这些变量具有三个可能的值，其区别如下：

- 当变量设置为时`nulls_equal`，所有`NULL`值都被视为相同（即，它们全部形成一个值组）。

  如果`NULL`值组的大小比平均非`NULL`值组的大小大得多，则此方法会使平均值组的大小向上倾斜。这使得索引在优化器中似乎没有那么有用，而对于查找非`NULL`值的联接而言，索引的作用实际上没有那么大。因此，该 `nulls_equal`方法可能导致优化器`ref`在应该使用索引时不使用索引 。

- 当变量设置为时 `nulls_unequal`，`NULL` 值将被认为是不同的。而是，每个 `NULL`值形成一个单独的大小为1的值组。

  如果您有很多`NULL`值，则此方法会使平均值组的大小向下倾斜。如果平均非`NULL`值组大小很大，则将`NULL`每个值作为一组大小1进行计数会使优化器高估寻找非`NULL` 值的联接的索引值。因此，当其他方法可能更好时，该`nulls_unequal` 方法可能导致优化器将此索引用于 `ref`找。

- 将变量设置为时 `nulls_ignored`，`NULL` 将忽略值。

如果您倾向于使用许多使用`<=>`而不是的联接 `=`，则 `NULL`值在比较中并不特殊，并且一个`NULL`等于另一个。在这种情况下，`nulls_equal`是适当的统计方法。

该`innodb_stats_method`系统变量具有全局值; 该 `myisam_stats_method`系统变量有全局和会话值。设置全局值会影响从相应存储引擎收集表的统计信息。设置会话值只会影响当前客户端连接的统计信息收集。这意味着您可以通过将会话值设置为来强制使用给定的方法重新生成表的统计信息，而不会影响其他客户端 `myisam_stats_method`。

要重新生成`MyISAM`表统计信息，可以使用以下任何一种方法：

- 执行**myisamchk --stats_method = \*`method_name`\* --analyze**
- 更改表以使其统计信息过时（例如，插入一行然后将其删除），然后设置 `myisam_stats_method`并发出一条`ANALYZE TABLE` 语句

有关使用`innodb_stats_method`和的 一些注意事项 `myisam_stats_method`：

- 如前所述，您可以强制显式收集表统计信息。但是，MySQL可能还会自动收集统计信息。例如，如果在执行表语句的过程中，其中一些语句修改了表，则MySQL可能会收集统计信息。（例如，这可能发生在批量插入或删除操作或某些 `ALTER TABLE`语句中。）如果发生这种情况，则使用任何值 `innodb_stats_method`或 `myisam_stats_method`当时有。因此，如果您使用一种方法收集统计信息，但是稍后稍后自动收集表的统计信息时，系统变量设置为另一种方法，则将使用另一种方法。
- 无法确定使用哪种方法为给定表生成统计信息。
- 这些变量仅适用于`InnoDB`和 `MyISAM`表。其他存储引擎只有一种收集表统计信息的方法。通常它更接近该`nulls_equal`方法。