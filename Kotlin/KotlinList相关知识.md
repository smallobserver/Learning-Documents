#  Kotlin List 

### 常用函数

### filter

`filter`就像其本意一样，可以通过 filter 对 Kotlin list 进行过滤。例子如下（我们可以直接在 Kotlin Playground 中运行）：

```kotlin
fun main(){
    val numbers = listOf(1, -2, 3, -4, 5, -6)
    val positives = numbers.filter { x -> x > 0 }
    val negatives = numbers.filter { it < 0 }      // 这里我们可以使用 it 
	println("positive values: ${positives}")        // 打印 positive values: [1, 3, 5]
	println("negative values: ${negatives}")        // 打印 positive values: [-2, -4, -6]
}
```


### map

`map`能够将变化应用于集合中的所有元素。

```kotlin
fun main(){
    val numbers = listOf(1, -2, 3, -4, 5, -6)
    val positives = numbers.filter { x -> x > 0 }
    val negatives = numbers.filter { it < 0 }      // 这里我们可以使用 it 
    
    println("positive values: ${positives}")        // 打印 positive values: [1, 3, 5]
    println("negative values: ${negatives}")        // 打印 positive values: [-2, -4, -6]
}

```


### count

`count` 函数返回集合中的元素总数或与给定条件匹配的元素数。

```kotlin
fun main(){
    val numbers = listOf(1, -2, 3, -4, 5, -6)
    val totalCount = numbers.count()                     
    val evenCount = numbers.count { it % 2 == 0 }        
    
    println("totalCount: ${totalCount}")    // 打印 totalCount: 6
    println("evenCount: ${evenCount}")      // 打印 evenCount: 3
}

```


### first, last

`first`和`last`分别返回列表中第一个或最后一个元素的值。

```kotlin
fun main(){
    val numbers = listOf(1, -2, 3, -4, 5, -6)            
    val first = numbers.first()                          
    val last = numbers.last()                            
    val firstEven = numbers.first { it % 2 == 0 }        
    val lastOdd = numbers.last { it % 2 != 0 }                  
    
    println("first element: ${first}")          // 打印 first element: 1
    println("last element: ${last}")            // 打印 last element: -6
    println("first Even element: ${firstEven}") // first Even element: -2
    println("last Odd element: ${lastOdd}")     // last Odd element: 5
}

```


### any, all, none

这些函数检查是否存在与给定条件匹配的集合元素，并返回布尔值。

```kotlin
fun main(){
    val numbers = listOf(1, -2, 3, -4, 5, -6)            
    val anyNegative = numbers.any { it < 0 }             
    val anyGT6 = numbers.any { it > 6 }                  
    val allEven = numbers.all { it % 2 == 0 }            
    val allLess6 = numbers.all { it < 6 }  
    val allEven = numbers.none { it % 2 == 1 }           
    val allLess6 = numbers.none { it > 6 }               

    println("any negative elements: ${anyNegative}")    // 打印 any negative elements: true
    println("any elements larger than six: ${anyGT6}")  // any elements larger than six: false
    println("是否所有元素都是双数：${allEven}")           // 是否所有元素都是双数：false
    println("是否所有元素都小于6: ${allLess6}")           // 是否所有元素都小于6: true
}

```


### find，findLast

`find` 和 `findLast` 函数返回与给定条件匹配的第一个或最后一个集合元素。如果没有这样的元素，函数将返回 null。

```kotlin
fun main(){
    val words = listOf("Lets", "find", "something", "in", "collection", "somehow")  
    val first = words.find { it.startsWith("some") }           
    val last = words.findLast { it.startsWith("some") }     
    val nothing = words.find { it.contains("nothing") } 

    println("find first word which contains 'some': ${first}")
    println("find last word which contains 'some: ${last}")
    println("find first word which contains 'nothing': ${nothing}")
}

```
