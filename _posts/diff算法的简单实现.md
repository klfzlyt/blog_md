---
title: diff算法的简单实现
date: 2017-06-14 17:20:52
tags: 算法
---


```javascript
let diffchange = diffList([{ key: 1 }, { key: 2 }], [{ key: 33 }, { key: 1 }, { key: 3 }], 'key');
console.log('diffchange', diffchange);
```

![img](http://p1.meituan.net/dpgroup/0fe09be029636b209f5c40ac9f14484c128522.jpg)

 
```javascript
第一步 add：[{key:33},{ key: 1 }, { key: 2 }]
第二步 add：[{key:33},{ key: 1 },{ key: 3 }, { key: 2 }],
第三步 remove：[{key:33},{ key: 1 },{ key: 3 }]
```
<!--more-->
```javascript
//这个list要一个key,如果没有key,则移除所有的旧的元素，增加新的元素
function diffList(oldList, newList, key) {
	var move = function(list, from, to) {
		var item = list.splice(from, 1);
		if (from > to) list.splice(to + 1, 0, item[0]);
		else list.splice(to, 0, item[0]);
	};
	let changes = [];
	let lastIndex = 0;
	let copy_oldList = _.cloneDeep(oldList);
	for (var index = 0; index < newList.length; index++) {
		var elementKeyValue = newList[index][key];
		if (elementKeyValue) {
			let findedIndex = _.findIndex(oldList, { [key]: elementKeyValue });
			if (findedIndex == -1) {
				//老集合没有这个key的值
				//增加

				changes.push({
					type: 'add',
					index: index,
					item: newList[index],
				});
				//插入copyed
				copy_oldList.splice(index, 0, newList[index]);
			} else {
				//老集合有这个object,key一样
				//移动
				if (findedIndex >= lastIndex) {
					//not move
				} else {
					//move
					let item = oldList[findedIndex];
					let findedIndexCopyList = _.findIndex(copy_oldList, { [key]: item[key] });
					// changes.push({
					// 	type: 'move',
					// 	from: findedIndexCopyList,
					// 	to: index,
					// });
					/*
						var item = list.splice(from, 1);
						if (from > to) list.splice(to + 1, 0, item[0]);
						else list.splice(to, 0, item[0]);
					*/
					//删除from  插入 to
					changes.push({
						type: 'remove',
						index: findedIndexCopyList,
					});
					changes.push({
						type: 'add',
						index: index,
						item: item,
					});
					move(copy_oldList, findedIndexCopyList, index);
				}
				lastIndex = Math.max(lastIndex, findedIndex);
			}
		} else {
			//没有key的话 强行加入
			changes.push({
				type: 'add',
				index: index,
				item: newList[index],
			});
			copy_oldList.splice(index, 0, newList[index]);
		}
	}

	for (var indexDelte = 0; indexDelte < oldList.length; indexDelte++) {
		var oldItem = oldList[indexDelte];
		if (!oldItem[key]) {
			//如果没有key

			changes.push({
				type: 'remove',
				index: newList.length+indexDelte
			});
			continue;
		}
		let findedIndex = _.findIndex(newList, { [key]: oldItem[key] });
		if (findedIndex == -1) {
			//新的没找到，说明这个老的要删除掉
			changes.push({
				type: 'remove',
				index: _.findIndex(copy_oldList, { [key]: oldItem[key] }),
			});
			let findedIndex = _.findIndex(copy_oldList, { [key]: oldItem[key] });
			copy_oldList.splice(findedIndex, 1);
		}
	}
	return changes;
}

//let diffchage = diffList([{ key: 1 }, { key: 2 }], [{ key: 2 }, { key: 1 }, { key: 3 }], 'key');
//没有key的情况
//let diffchage = diffList([{ key: 1 }, { key: 2 }], [{ key: 33 }, { key: 1 }, { key: 3 }], 'key111');

let diffchange = diffList([{ key: 1 }, { key: 2 }], [{ key: 33 }, { key: 1 }, { key: 3 }], 'key');
console.log('diffchange', diffchange);
```