## 前言

在人们的印象中，可能觉得只有做后端的小伙伴才会接触到数据库。其实在前端的领域里面也有数据库，只是可能用的比较少，因为前端存储方案有很多，比如cookie、sessionstorage等等。

在浏览器上有两种数据库：webSQL和IndexedDB。但是如果在浏览器上需要用到数据库一般会使用Indexed DB数据库，webSQL基本上已经废弃了，具体原因小伙伴可以下来自己查查，今天主要就讲解Indexed DB数据库的使用。

## 1.IndexedDB简介

MDN官网是这样解释Indexed DB的：

> IndexedDB 是一种底层 API，用于在客户端存储大量的结构化数据（也包括文件/二进制大型对象（blobs））。该 API 使用索引实现对数据的高性能搜索。虽然 Web Storage 在存储较少量的数据很有用，但对于存储更大量的结构化数据来说力不从心。而 IndexedDB 提供了这种场景的解决方案。

官网上的这句话也很简单明了，意思就是IndexedDB主要用来客户端存储大量数据而生的，我们都知道cookie、localstorage等存储方式都有存储大小限制。如果数据量很大，且都需要客户端存储时，那么就可以使用IndexedDB数据库。

客户端各存储方式对比：

![](https://pic1.zhimg.com/80/v2-62ccd803cfefa77934cb96c3c4d6fd14_720w.jpg)## 2.IndexedDB使用场景

所有的场景都基于客户端需要存储大量数据的前提下：

1. 数据可视化等界面，大量数据，每次请求会消耗很大性能。
2. 即时聊天工具，大量消息需要存在本地。
3. 其它存储方式容量不满足时，不得已使用IndexedDB

## 3.IndexedDB特点

**（1） 非关系型数据库(NoSql)**

我们都知道MySQL等数据库都是关系型数据库，它们的主要特点就是数据都以一张二维表的形式存储，而Indexed DB是非关系型数据库，主要以键值对的形式存储数据。

**（2）持久化存储**

cookie、localStorage、sessionStorage等方式存储的数据当我们清楚浏览器缓存后，这些数据都会被清除掉的，而使用IndexedDB存储的数据则不会，除非手动删除该数据库。

**（3）异步操作**

IndexedDB操作时不会锁死浏览器，用户依然可以进行其他的操作，这与localstorage形成鲜明的对比，后者是同步的。

**（4）支持事务**

IndexedDB支持事务(transaction)，这意味着一系列的操作步骤之中，只要有一步失败了，整个事务都会取消，数据库回滚的事务发生之前的状态，这和MySQL等数据库的事务类似。

**（6）同源策略**

IndexedDB同样存在同源限制，每个数据库对应创建它的域名。网页只能访问自身域名下的数据库，而不能访问跨域的数据库。

**（7）存储容量大**

这也是IndexedDB最显著的特点之一了，这也是不用localStorage等存储方式的最好理由。

## 4.IndexedDB重要概念讲解

### 4.1仓库objectStore

IndexedDB没有表的概念，它只有仓库store的概念，大家可以把仓库理解为表即可，即一个store是一张表。

### 4.2索引index

在关系型数据库当中也有索引的概念，我们可以给对应的表字段添加索引，以便加快查找速率。在IndexedDB中同样有索引，我们可以在创建store的时候同时创建索引，在后续对store进行查询的时候即可通过索引来筛选，给某个字段添加索引后，在后续插入数据的过成功，索引字段便不能为空。

### 4.3游标cursor

游标是IndexedDB数据库新的概念，大家可以把游标想象为一个指针，比如我们要查询满足某一条件的所有数据时，就需要用到游标，我们让游标一行一行的往下走，游标走到的地方便会返回这一行数据，此时我们便可对此行数据进行判断，是否满足条件。

【注意】：IndexedDB查询不像MySQL等数据库方便，它只能通过主键、索引、游标方式查询数据。

### 4.4事务

IndexedDB支持事务，即对数据库进行操作时，只要失败了，都会回滚到最初始的状态，确保数据的一致性。

## 5.IndexedDB实操

IndexedDB所有针对仓库的操作都是基于事务的。

### 5.1创建或连接数据库

**代码如下：**

```js
/**
 * 打开数据库
 * @param {object} dbName 数据库的名字
 * @param {string} storeName 仓库名称
 * @param {string} version 数据库的版本
 * @return {object} 该函数会返回一个数据库实例
 */
function openDB(dbName, version = 1) {
  return new Promise((resolve, reject) => {
    //  兼容浏览器
    var indexedDB =
      window.indexedDB ||
      window.mozIndexedDB ||
      window.webkitIndexedDB ||
      window.msIndexedDB;
    let db;
    // 打开数据库，若没有则会创建
    const request = indexedDB.open(dbName, version);
    // 数据库打开成功回调
    request.onsuccess = function (event) {
      db = event.target.result; // 数据库对象
      console.log("数据库打开成功");
      resolve(db);
    };
    // 数据库打开失败的回调
    request.onerror = function (event) {
      console.log("数据库打开报错");
    };
    // 数据库有更新时候的回调
    request.onupgradeneeded = function (event) {
      // 数据库创建或升级的时候会触发
      console.log("onupgradeneeded");
      db = event.target.result; // 数据库对象
      var objectStore;
      // 创建存储库
      objectStore = db.createObjectStore("signalChat", {
        keyPath: "sequenceId", // 这是主键
        // autoIncrement: true // 实现自增
      });
      // 创建索引，在后面查询数据的时候可以根据索引查
      objectStore.createIndex("link", "link", { unique: false }); 
      objectStore.createIndex("sequenceId", "sequenceId", { unique: false });
      objectStore.createIndex("messageType", "messageType", {
        unique: false,
      });
    };
  });
}
```

我们将创建数据库的操作封装成了一个函数，并且该函数返回一个promise对象，使得在调用的时候可以链式调用，函数主要接收两个参数：数据库名称、数据库版本。函数内部主要有三个回调函数，分别是：

* **onsuccess**：数据库打开成功或者创建成功后的回调，这里我们将数据库实例返回了出去。
* **onerror**：数据库打开或创建失败后的回调。
* **onupgradeneeded**：当数据库版本有变化的时候会执行该函数，比如我们想创建新的存储库（表），就可以在该函数里面操作，更新数据库版本即可。

### 5.2插入数据

```js
/**
 * 新增数据
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {string} data 数据
 */
function addData(db, storeName, data) {
  var request = db
    .transaction([storeName], "readwrite") // 事务对象 指定表格名称和操作模式（"只读"或"读写"）
    .objectStore(storeName) // 仓库对象
    .add(data);

  request.onsuccess = function (event) {
    console.log("数据写入成功");
  };

  request.onerror = function (event) {
    console.log("数据写入失败");
  };
}
```

IndexedDB插入数据需要通过事务来进行操作，插入的方法也很简单，利用IndexedDB提供的add方法即可，这里我们同样将插入数据的操作封装成了一个函数，接收三个参数，分别如下：

* db：在创建或连接数据库时，返回的db实例，需要那个时候保存下来。
* storeName：仓库名称(或者表名)，在创建或连接数据库时我们就已经创建好了仓库。
* data：需要插入的数据，通常是一个对象

**【注意】：**插入的数据是一个对象，而且必须包含我们声明的索引键值对。

### 5.3通过主键读取数据

**代码如下：**

```js
/**
 * 通过主键读取数据
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {string} key 主键值
 */
function getDataByKey(db, storeName, key) {
  return new Promise((resolve, reject) => {
    var transaction = db.transaction([storeName]); // 事务
    var objectStore = transaction.objectStore(storeName); // 仓库对象
    var request = objectStore.get(key); // 通过主键获取数据

    request.onerror = function (event) {
      console.log("事务失败");
    };

    request.onsuccess = function (event) {
      console.log("主键查询结果: ", request.result);
      resolve(request.result);
    };
  });
}
```

主键即刚刚我们在创建数据库时声明的keyPath，通过主键只能查询出一条数据。

### **5.4通过游标查询数据**

**代码如下：**

```js
/**
 * 通过游标读取数据
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 */
function cursorGetData(db, storeName) {
  let list = [];
  var store = db
    .transaction(storeName, "readwrite") // 事务
    .objectStore(storeName); // 仓库对象
  var request = store.openCursor(); // 指针对象
  // 游标开启成功，逐行读数据
  request.onsuccess = function (e) {
    var cursor = e.target.result;
    if (cursor) {
      // 必须要检查
      list.push(cursor.value);
      cursor.continue(); // 遍历了存储对象中的所有内容
    } else {
      console.log("游标读取的数据：", list);
    }
  };
}
```

上面函数开启了一个游标，然后逐行读取数据，存入数组，最终得到整个仓库的所有数据。

### 5.5通过索引查询数据

**代码如下：**

```js
/**
 * 通过索引读取数据
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {string} indexName 索引名称
 * @param {string} indexValue 索引值
 */
function getDataByIndex(db, storeName, indexName, indexValue) {
  var store = db.transaction(storeName, "readwrite").objectStore(storeName);
  var request = store.index(indexName).get(indexValue);
  request.onerror = function () {
    console.log("事务失败");
  };
  request.onsuccess = function (e) {
    var result = e.target.result;
    console.log("索引查询结果：", result);
  };
}
```

索引名称即我们创建仓库的时候创建的索引名称，也就是键值对中的键，最终会查询出所有满足我们传入函数索引值的数据。

### **5.6通过索引和游标查询数据**

通过5.4节和5.5节我们发现，单独通过索引或者游标查询出的数据都是部分或者所有数据，如果我们想要查询出索引中满足某些条件的所有数据，那么单独使用索引或游标是无法实现的。当然，你也可以查询出所有数据之后在循环数组筛选出合适的数据，但是这不是最好的实现方式，最好的方式当然是将索引和游标结合起来。

**代码如下：**

```js
/**
 * 通过索引和游标查询记录
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {string} indexName 索引名称
 * @param {string} indexValue 索引值
 */
function cursorGetDataByIndex(db, storeName, indexName, indexValue) {
  let list = [];
  var store = db.transaction(storeName, "readwrite").objectStore(storeName); // 仓库对象
  var request = store
    .index(indexName) // 索引对象
    .openCursor(IDBKeyRange.only(indexValue)); // 指针对象
  request.onsuccess = function (e) {
    var cursor = e.target.result;
    if (cursor) {
      // 必须要检查
      list.push(cursor.value);
      cursor.continue(); // 遍历了存储对象中的所有内容
    } else {
      console.log("游标索引查询结果：", list);
    }
  };
  request.onerror = function (e) {};
}
```

上面函数接收四个参数，分别是：

* db：数据库实例
* storeName：仓库名
* indexName：索引名称
* indexName：索引值

利用索引和游标结合查询，我们可以查询出索引值满足我们传入函数值的所有数据对象，而不是之查询出一条数据或者所有数据。

### 5.7通过索引和游标分页查询

IndexedDB分页查询不像MySQL分页查询那么简单，没有提供现成的API，如limit等，所以需要我们自己实现分页。

**代码如下：**

```js
/**
 * 通过索引和游标分页查询记录
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {string} indexName 索引名称
 * @param {string} indexValue 索引值
 * @param {number} page 页码
 * @param {number} pageSize 查询条数
 */
function cursorGetDataByIndexAndPage(
  db,
  storeName,
  indexName,
  indexValue,
  page,
  pageSize
) {
  let list = [];
  let counter = 0; // 计数器
  let advanced = true; // 是否跳过多少条查询
  var store = db.transaction(storeName, "readwrite").objectStore(storeName); // 仓库对象
  var request = store
    .index(indexName) // 索引对象
    .openCursor(IDBKeyRange.only(indexValue)); // 指针对象
  request.onsuccess = function (e) {
    var cursor = e.target.result;
    if (page > 1 && advanced) {
      advanced = false;
      cursor.advance((page - 1) * pageSize); // 跳过多少条
      return;
    }
    if (cursor) {
      // 必须要检查
      list.push(cursor.value);
      counter++;
      if (counter < pageSize) {
        cursor.continue(); // 遍历了存储对象中的所有内容
      } else {
        cursor = null;
        console.log("分页查询结果", list);
      }
    } else {
      console.log("分页查询结果", list);
    }
  };
  request.onerror = function (e) {};
}
```

这里用到了IndexedDB的一个API：advance。该函数可以让我们的游标跳过多少条开始查询。假如我们的额分页是每页10条数据，现在需要查询第2页，那么我们就需要跳过前面10条数据，从11条数据开始查询，直到计数器等于10，那么我们就关闭游标，结束查询。

### 5.8更新数据

IndexedDB更新数据较为简单，直接使用put方法，值得注意的是如果数据库中没有该条数据，则会默认增加该条数据，否则更新。有些小伙伴喜欢更新和新增都是用put方法，这也是可行的。

**代码如下：**

```js
/**
 * 更新数据
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {object} data 数据
 */
function updateDB(db, storeName, data) {
  var request = db
    .transaction([storeName], "readwrite") // 事务对象
    .objectStore(storeName) // 仓库对象
    .put(data);

  request.onsuccess = function () {
    console.log("数据更新成功");
  };

  request.onerror = function () {
    console.log("数据更新失败");
  };
}
```

put方法接收一个数据对象。

### 5.9通过主键删除数据

主键即我们创建数据库时申明的keyPath，它是唯一的。

**代码如下：**

```js
/**
 * 通过主键删除数据
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {object} id 主键值
 */
function deleteDB(db, storeName, id) {
  var request = db
    .transaction([storeName], "readwrite")
    .objectStore(storeName)
    .delete(id);

  request.onsuccess = function () {
    console.log("数据删除成功");
  };

  request.onerror = function () {
    console.log("数据删除失败");
  };
}
```

该种删除只能删除一条数据，必须传入主键。

### 5.10通过索引和游标删除指定数据

有时候我们拿不到主键值，只能只能通过索引值来删除，通过这种方式，我们可以删除一条数据（索引值唯一）或者所有满足条件的数据（索引值不唯一）。

**代码如下：**

```js
/**
 * 通过索引和游标删除指定的数据
 * @param {object} db 数据库实例
 * @param {string} storeName 仓库名称
 * @param {string} indexName 索引名
 * @param {object} indexValue 索引值
 */
function cursorDelete(db, storeName, indexName, indexValue) {
  var store = db.transaction(storeName, "readwrite").objectStore(storeName);
  var request = store
    .index(indexName) // 索引对象
    .openCursor(IDBKeyRange.only(indexValue)); // 指针对象
  request.onsuccess = function (e) {
    var cursor = e.target.result;
    var deleteRequest;
    if (cursor) {
      deleteRequest = cursor.delete(); // 请求删除当前项
      deleteRequest.onerror = function () {
        console.log("游标删除该记录失败");
      };
      deleteRequest.onsuccess = function () {
        console.log("游标删除该记录成功");
      };
      cursor.continue();
    }
  };
  request.onerror = function (e) {};
}
```

上段代码可以删除索引值为indexValue的所有数据，值得注意的是使用了[IDBKeyRange.only(）](https://link.zhihu.com/?target=https%3A//developer.mozilla.org/zh-CN/docs/Web/API/IDBKeyRange)API，该API代表只能当两个值相等时，具体API解释可参考MDN官网。

### 5.11关闭数据库

当我们数据库操作完毕后，建议关闭它，节约资源。

**代码如下：**

```js
/**
 * 关闭数据库
 * @param {object} db 数据库实例
 */
function closeDB(db) {
  db.close();
  console.log("数据库已关闭");
}
```

5.12删除数据库

最后我们需要删库跑路，删除操作也很简单。

**代码如下：**

```js
/**
 * 删除数据库
 * @param {object} dbName 数据库名称
 */
function deleteDBAll(dbName) {
  console.log(dbName);
  let deleteRequest = window.indexedDB.deleteDatabase(dbName);
  deleteRequest.onerror = function (event) {
    console.log("删除失败");
  };
  deleteRequest.onsuccess = function (event) {
    console.log("删除成功");
  };
}
```

## 总结

IndexedDB数据库没有我们想象的那么复杂，了解了它的几个基本概念，上手还是很快的，无非就是增删改查等等，虽然可能开发中用的少，但是了解一下不至于真正用到的时候两眼抓瞎。
