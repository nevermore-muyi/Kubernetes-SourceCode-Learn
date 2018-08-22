### MongoDB聚合

#### 1.$match

```
$match类似于SQL中的where，一般建议用在最初查询时，形如：{$match:{"":""}}
```

#### 2.$project

```
$project用于提取某些字段，也可以重命名字段，如：
db.userEntity.aggregate({$match:{"userName":"name9000"}},{"$project":{"name":"$userName","userName":1}})
除了简单的提取和排除，还支持各种表达式，如：
数学表达式：   $add $subtract $multiply $divide $mod
日期表达式:    $year $month $week $datOfMonth $dayOfWeek $dayOfYear $hour $minute $second
字符串表达式:  $substr $concat $toLower $toUpper
逻辑表达式：   $cmp $cond $ifNull
```

#### 3.$group

```
$group可以将文档按照特定的字段进行分组，需要将选定的字段赋值给"_id"，形如：{$group:{"_id":"$day"}}
分组之后，可以通过特定的操作符做一些操作
算术操作符： $sum $avg
极值操作符： $max $min $first $last
数组操作符： $addToSet(不重复) $push(随意添加)
```

#### 4.$unwind

```
$unwind可以将原集合数组中的每一个值拆分为单独的文档，形如：{$unwind:"$comments"}
```

#### 5.$sort

```
$sort进行排序，建议在管道的第一阶段进行排序
```

#### 6.$limit

#### 7.$skip