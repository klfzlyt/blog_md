---
title: 2个字符串的编辑距离（Levenshtein_distance）
date: 2017-06-04 21:20:52
tags: 算法 动态规划
---

在react中，在render后的diff阶段，对一个列表进行了diff比较，react的实现比较高效，是o(n)量级，一种通用的diff是类似于求两个字符串的编辑距离(o(m*n)),参考：
> https://en.wikipedia.org/wiki/Levenshtein_distance

比如要计算cafe和coffee的编辑距离，cafe→caffe→coffe→coffee，要经过3次转变。

动态规划解决思路：
<!--more-->

```javascript

/*
 Utils
*/
function replareStr(word1, pos, char) {
	let newWord = '';
	for (let i = 0; i < word1.length; i++) {
		if (i == pos) {
			newWord += char;
		} else {
			newWord += word1[i];
		}
	}
	return newWord;
}
function deleteStr(word1, pos) {
	let newWord = '';
	for (let i = 0; i < word1.length; i++) {
		if (i == pos) {
			// ddnewWord+=char;
		} else {
			newWord += word1[i];
		}
	}
	return newWord;
}
function insterStr(word1, pos, char) {
	let newWord = '';
	if (pos < 0) newWord += char;
	for (let i = 0; i < word1.length; i++) {
		newWord += word1[i];
		if (i == pos) {
			newWord += char;
		}
	}
	return newWord;
}




/*
 * 需要把word1 变到 word2 
 * @param {string} word1
 * @param {string} word2
 * @return {array} 表示从word1 改变到 word2 中间要过渡的字符串组
 */
var minDistance = function(word1, word2) {
	let rowNumber = word1.length + 1;
	let columnNumber = word2.length + 1;
	let DP = [];
	/*
    init dp
    */
	for (let i = 0; i < rowNumber; i++) {
		let temp = Array(columnNumber);
		if (i == 0) {
			for (let j = 0; j < columnNumber; j++) {
				temp[j] = j;
			}
		} else {
			temp[0] = i;
		}
		DP.push(temp);
	}
	for (let i = 1; i < rowNumber; i++) {
		for (let j = 1; j < columnNumber; j++) {
			let first, second, third;
			if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
				first = DP[i - 1][j - 1];
			} else {
				first = DP[i - 1][j - 1] + 1;
			}
			second = DP[i][j - 1] + 1;
			third = DP[i - 1][j] + 1;
			DP[i][j] = Math.min(first, second, third);
		}
	}
	let internalwords = [];
	let endRow = rowNumber - 1;
	let endColumn = columnNumber - 1;
	let numberOperation = DP[endRow][endColumn];

	while (numberOperation) {
		let isOperated = false;
		let duijiao = Infinity;
		let above = Infinity;
		let left = Infinity;
		if (endRow - 1 < 0) {
			duijiao = Infinity;
			above = Infinity;
		} else {
			if (endColumn - 1 >= 0) {
				duijiao = DP[endRow - 1][endColumn - 1];
			} else {
				duijiao = Infinity;
			}

			above = DP[endRow - 1][endColumn];
		}
		if (endColumn - 1 < 0) {
			left = Infinity;
		} else {
			left = DP[endRow][endColumn - 1];
		}
		let min = Math.min(duijiao, above, left);
		if (min == duijiao) {
			isOperated = true;
			endRow = endRow - 1;
			endColumn = endColumn - 1;
			if (duijiao < numberOperation) {
				//如果小于则替换，不小于不做操作
				let newword1 = word1;
				let tobechar = word2.charAt(endColumn);
				internalwords.push((word1 = replareStr(newword1, endRow, tobechar)));
			}
		} else if (min == above && !isOperated) {
			//删除above
			isOperated = true;
			endRow = endRow - 1;
			internalwords.push((word1 = deleteStr(word1, endRow)));
		} else if (min == left && !isOperated) {
			//向左添加
			isOperated = true;
			endColumn = endColumn - 1;
			let tobeinsertChar = word2.charAt(endColumn);
			internalwords.push((word1 = insterStr(word1, endRow - 1, tobeinsertChar)));
		}

		numberOperation = DP[endRow][endColumn];
	}
	return internalwords;
};

//let interwords = minDistance('sunday', 'saturday');
//let interwords = minDistance('sitting', 'kitten');
let interwords = minDistance('a12234', '1234');
console.log('interwords', interwords);
/*
	2步
	interwords (2) ["a1234", "1234"]
*/
```
从sunday——>saturday,需要3步["surday", "sturday", "saturday"]
从sitting——>kitten，需要3步["sittin", "sitten", "kitten"]
从a12234——>1234，需要2步["a1234", "1234"]
