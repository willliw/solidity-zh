# Solidity Assembly 

Assembly 語法可以不用搭配 Solidity 使用，同時也可以以 "inline assembly" 的方式嵌在在智能合約源碼裡。我們從怎麼使用 inline assembly 開始，之後會描述不同 assembly 寫法之間的差別。

## Inline Assembly
為了能夠更精確的描述智能合約的行為，Solidity 允許直接使用更接近底層虛擬機語法的 inline assembly 來描述合約行為。因為 EVM 是堆疊(stack)器，通常很難指出現在跑到堆疊的哪裡或是當前提供的參數是什麼。Solidity 提供的 inline assembly 試著解決上述以及提供以下特色時會遇到的問題：

- 函式型態的操作碼：```mul(1, add(2 ,3))``` ，而非 ```push 3 push 2 add push1 1 mul```
- assembly本地變數: ```let x := add(2, 3)  let y := mload(0x40)  x := add(x, y)```
- 讀取外部變數: ```function f(uint x) public { assembly { x := sub(x, 1)}}```
- 標籤: ```let x := 10 repeat: x := sub(x, 1)  jumpi(repeat, eq(x, 0))```
- 迴圈: ```for { let i := 0} lt(i, x) { i := add(i, 1) } { y := mul(2, y) }```
- if描述: ```if slt(x, 0) {x := sub(0, x)}```
- switch描述: ```switch x case 0 { y := mul(x, 2) } default { y := 0}```
- 函式呼叫: ```function f(x) -> y { switch x case 0 { y := 1} default { y := mul(x, f(sub(x, 1)))}}```

接下來介紹 inline assembly 細節。

```
Warning: 
Inline assembly 是直接跟底層虛擬機溝通的方式。會繞過很多 Solidity 提供的嚴重錯誤檢查。 
```

## 範例

底下的範例提供一個程式庫 (library)，該程式庫會讀取其他合約並把他載入一個```bytes```變數。這個功能無法指透過 Solidity 來完成，如果透過這個包括 assemlby 的程式庫可以讓 Solidity 更完整。

```
pragma solidity ^0.4.0;

library GetCode {
	function at(address _addr) public returns (bytes o_code) {
		assembly {
			// 抓該地址對應到合約的大小，這Solidity做不到
			let size := extcodesize(_addr)
			// 宣告輸出的bytes陣列，這用Solidity也能做
			// by using o_code = new bytes(size)
			o_code := mload(0x40)
			// 把新的記憶體結束位置加上位移給補上 
			mstore(0x40, add(o_code, and(add(add(size, 0x20), 0x1f))))
			// store length in memory 
			mstore(o_code, size)
			// actually retrieve the code, this need assembly
			extcodecopy(_addr, add(o_code, 0x20), 0, size)
		}
	}
}
```

在某些編譯器無法最佳化的狀況，也可以透過inline assembly來得到好處。請注意這是因為編譯器跳過很多檢查，所以相對的 assembly 也比較難寫。再用 assembly 描述複雜邏輯的時候請確保瞭解自己的需求。

```
pragma solidity ^0.4.12;

library VectorSum {
	// This function is less efficient because the optimizer currently fails to
    // remove the bounds checks in array access.
    function sumSolidity(uint[] _data) public returns (uint256 o_sum) {
    	for (uint i = 0; i < _data.length; ++i)
    		o_sum += _data[i];
    }

    // We know that we only access the array in bounds, so we can avoid the check.
    // 0x20 needs to be added to an array because the first slot contains the
    // array length.
    function sumAsm(uint[] _data) public returns (uint o_sum) {
    	for (uint i = 0; i < _data.length; ++i) {
    		assembly {
    			o_sum := add(o_sum, mload(add(add(_data, 0x20)), mul(i, 0x20)))
    		}
    	}
    } 

    // Same as above, but accomplish the entire code within inline assembly.
    function sumPureAsm(uint[] _data) public returns (uint o_sum) {
    	assembly {
    		// Load the length (first 32 bytes)
    		let len := mload(_data)

    		// Skip over the length field.
    		// 
    		// Keep temporary variable so it can be incremented in place. 
    		// 
    		// NOTE: incrementing _data would result in an unusable
    		//       _data variable after this assembly block
    		let data := add(_data, 0x20)

    		// Iterate until the bound is not met.
    		for 
    			{let end := add(data, len)}
    			lt(data, end)
    			{data := add(data, 0x20)}
    		{
    			o_sum := add(o_sum, mload(data))
    		}
    	}
    }

}
```

