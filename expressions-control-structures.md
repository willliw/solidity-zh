# 表達式和控制結構
[原文](http://solidity.readthedocs.io/en/latest/control-structures.html)

## 參數輸入 (Input Parameters)
參數的宣告跟一般的變數一樣。不過如果是用不到的參數，在宣告的時候可以直接忽略參數名。以下舉例的合約接受某個有兩個整數參數的外部指令，會寫成這樣：

```
pragma solidity ^0.4.16;

contract Simple {
	function taker(uint _a, uint _b) public pure {
		// do something with _a and _b.
	}
}
```

## 輸出變數 (Output Parameters)
輸出變數可以在 “returns" 關鍵字之後宣告。以下舉例的合約預期會回傳兩個值，分別是輸入的兩個整數的總和(sum)，以及兩個數的乘積(product)，會寫成這樣：

```
pragma solidity ^0.4.16;

contract Simple {
    function arithmetics(uint _a, uint _b)
        public
        pure
        returns (uint o_sum, uint o_product)
    {
        o_sum = _a + _b;
        o_product = _a * _b;
    }
}
```
輸出變數的名稱可以省略。回傳值也可以用"return"來寫。此外，return的寫法也可以直接回傳多個變數，詳見[Returning Multiple Values](http://solidity.readthedocs.io/en/latest/control-structures.html#multi-return). 回傳的內容(輸出變數)預設為0, 如果在函式執行期間沒有設定到該值，會直接回傳0回去。

輸入參數以及輸出的變數可以直接在函式裡使用。所以也可以放在式子的左手邊直接賦值。

## 控制結構 (Control Structures)

除了switch和goto, Javascript 裡的控制語法都可以在 Solidity 裡使用。分別是：if, else, while, do, for, break, continue, return, ? :, 其語義均和C / JavaScript一樣。

條件語句中的括號不能省略,但在單條語句前後的大括號可以省略。

注意,（Solidity中）沒有像C/JavaScrip那樣 ，從非布靈類型到布靈類型的轉換, 所以if (1){…}不是合法的語句。

## 回傳多個變數 (Returning Multiple Values)

當某個函式回傳多個變數，可以用 return (v0, v1, ..., vn) 來作回傳。括號裡變數的數量一定要跟宣告輸出變數的數量一致。

## 函式呼叫

內部函式呼叫

同個合約裡的函式可以直接呼叫(內部函式),遞歸也可以, 所以以下的例子是合法的:

```
pragma solidity ^0.4.16;

contract C {
    function g(uint a) public pure returns (uint ret) { return f(); }
    function f() internal pure returns (uint ret) { return g(7) + f(); }
}
```

這些函式呼叫會在EVM裡被編譯成jumps語句。在遞迴呼叫函示的時候，當前內存(memory)不會被清空,意思是可以透過內存直接傳送索引(reference)，(比起外部呼叫)使得效率較高。只有同一個合約的函式可在內部被呼叫。

外部函式呼叫

表達式this.g(8);或是c.g(2); (其中c是一個合約的位址)也是一個合法的函式呼叫。與前面不同的是，這個函式被稱爲“外部”的, 通過交易呼叫(message call),而不是直接通過jumps來呼叫。在建構子(constructor)裡無法使用 this 關鍵字，因為在建構子未完成的時候合約不算是創造完成。

若要使用其他合約的函式也需要透過外部韓式的呼叫。對於一個外部函式的呼叫,這個函式的所有參數都會被複制到內存中。

當呼叫其他合約的函式時, 若需要附上一些 Ether (單位為 Wei)，或是要設定gas可以透過 .value() 跟 .gas()，舉例如下：

```
pragma solidity ^0.4.0;

contract InfoFeed {
    function info() public payable returns (uint ret) { return 42; }
}

contract Consumer {
    InfoFeed feed;
    function setFeed(address addr) public { feed = InfoFeed(addr); }
    function callFeed() public { feed.info.value(10).gas(800)(); }
}
```

在上面的例子裡，info 函式必須要用 payable 關鍵字來修飾。否則的話 .value() 不會被接受。

另外，表達式 InfoFeed(addr) 是一個明確的類型轉換 (explicit type conversion)， 意思是“我們知道給定的地址的合約類型是InfoFeed”, 如此就不會執行該合約的建構子。 在使用類型轉換的時候，必須要確定該合約類型，千萬不要在不確定該合約類型的時候作轉換。

我們也可以直接使用函數setFeed(InfoFeed _feed) { feed = _feed; }。 注意： feed.info.value(10).gas(800) 只是先在本地端設定 value 和 gas 的值， 實際的呼叫為最後的括號結束後才執行。

在做函式呼叫的時候，若被呼叫的合約不存在或是呼叫的函式用遇到異常(exceptions)或用完了所有gas (run out of gas) 等，會造成異常狀況。

## 具名呼叫和匿名函式參數

在呼叫函式的時候，也可以透過 {} ，把參數的名字跟值不計順序傳遞過去，範例如下。
使用的名字必須跟函式所宣告得一模一樣，但可以不用照順序放。

```
pragma solidity ^0.4.0;

contract C {
    function f(uint key, uint value) public {
        // ...
    }

    function g() public {
        // named arguments
        f({value: 2, key: 3});
    }
}
```

## 忽略函式參數的名字

沒有用到的參數(尤其是回傳的參數)可以直接忽略。這些參數依然會被放進堆疊 (stack) 裡，但無法被存取。

```
pragma solidity ^0.4.16;

contract C {
    // omitted name for parameter
    function func(uint k, uint) public pure returns(uint) {
        return k;
    }
}
```

## 使用 new 關鍵字新建合約
可以用 new 關鍵來新建合約。該合約程式碼需要在新建之前就知道，所以不可能會出現遞迴新建合約的狀況發生。

```
pragma solidity ^0.4.0;

contract D {
    uint x;
    function D(uint a) public payable {
        x = a;
    }
}

contract C {
    D d = new D(4); // will be executed as part of C's constructor

    function createD(uint arg) public {
        D newD = new D(arg);
    }

    function createAndEndowD(uint arg, uint amount) public payable {
        // Send ether along with the creation
        D newD = (new D).value(amount)(arg);
    }
}
```

從上面的例子裡可以看到，在新建一個 D 合約的執行緒 (instance) 可以透過 .value() 給他一些 Ether，但無法設定 gas 的使用量。如果新建合約失敗 (可能原因有堆疊不夠，餘額不足等)，會產生一個異常 (exception)。

## 表達式計算的次序 (Order of Evaluation if Expressions)

表達式的計算順序是不確定的（準確地說是, 順序表達式樹中的子節點表達式計算順序是不確定的, 但他們對節點本身，計算表達式順序當然是確定的)。只保證語句執行順序，以及布爾表達式的短路規則。

## 賦值 (Assignment)

析構賦值並返回多個值 (Destructuring Assignment and Returning Multiple Values)

Solidity 內部允許元組(tuple)類型，即一系列在編譯時大小為常量的不同類型的變數。這些元組可以用來同時返回多個值，並且同時將它們分配給多個變量(或左值運算)

```
pragma solidity ^0.4.16;

contract C {
    uint[] data;

    function f() public pure returns (uint, bool, uint) {
        return (7, true, 2);
    }

    function g() public {
        // Declares and assigns the variables. Specifying the type explicitly is not possible.
        var (x, b, y) = f();
        // Assigns to a pre-existing variable.
        (x, y) = (2, 7);
        // Common trick to swap values -- does not work for non-value storage types.
        (x, y) = (y, x);
        // Components can be left out (also for variable declarations).
        // If the tuple ends in an empty component,
        // the rest of the values are discarded.
        (data.length,) = f(); // Sets the length to 7
        // The same can be done on the left side.
        (,data[3]) = f(); // Sets data[3] to 2
        // Components can only be left out at the left-hand-side of assignments, with
        // one exception:
        (x,) = (1,);
        // (1,) is the only way to specify a 1-component tuple, because (1) is
        // equivalent to 1.
    }
}
```

## 陣列和結構體的組合 (Complications for Arrays and Structs)

對於陣列和結構體這樣的非值類型，賦值的語義更復雜些。賦值到一個狀態變數總是會複製一個新的獨立副本。而如果是賦值到一個局部變數，只需要直接複製到一個 32 bytes 大小的靜態類別。
如果一個結構或陣列(包括 bytes 和 String)從狀態變數被賦值到局部變數，這個局部變數只需保存原始狀態變數的引用 (reference)。第二次賦值到局部變數不會修改狀態，只會修改到引用的位置。賦值到局部變數的成員(或元素)會改變到智能合約的狀態變數。

## 作用域和宣告 (Scoping and Declarations)

當一個變數被宣告的時候，不管其型態為何，會預設一個全部都是0的初始值。被稱為"零狀態" (zero-state)。舉例來說，bool 的預設值為 false，uint 或是 int 的預設初始值為 0。如果是固定長度的陣列或是 bytes1 到 bytes32，每個元素的初始值會跟該類型一樣。而如果是動態長度的陣列，像是 bytes 和 string，其預設的初始值為空陣列或字串。

不同於大部分語言的作用域規則，Solidity 繼承 JavaScript 的規則，使得在函式裡任意位置宣告的變數可以讓整個函式使用。如此，以下的例子會造成編譯錯誤 (Idenrifier already declared):

```
// This will not compile

pragma solidity ^0.4.16;

contract ScopingErrors {
    function scoping() public {
        uint i = 0;

        while (i++ < 1) {
            uint same1 = 0;
        }

        while (i++ < 2) {
            uint same1 = 0;// Illegal, second declaration of same1
        }
    }

    function minimalScoping() public {
        {
            uint same2 = 0;
        }

        {
            uint same2 = 0;// Illegal, second declaration of same2
        }
    }

    function forLoopScoping() public {
        for (uint same3 = 0; same3 < 1; same3++) {
        }

        for (uint same3 = 0; same3 < 1; same3++) {// Illegal, second declaration of same3
        }
    }
}
```

另外，如果有一個變數宣告，該變數會在函式一開始就被初始化。如此，以下的例子雖然寫的不是很好，但卻是合法的：

```
pragma solidity ^0.4.0;

contract C {
    function foo() public pure returns (uint) {
        // baz is implicitly initialized as 0
        uint bar = 5;
        if (true) {
            bar += baz;
        } else {
            uint baz = 10;// never executes
        }
        return bar;// returns 5
    }
}
```

## 錯誤處理：Assert, Require, revert 和異常

在做錯誤處理的時候， solidity 用狀態回復這個異常(state-reverting exceptions)來處理。 這個異常會回復所有對狀態的改變，並對發出交易的對象標示錯誤。assert 和 require 函式可以用來檢查條件，如果條件沒有滿足就會發出一個異常。assert 函式應只能被用來做合約內部的錯誤檢查，保證某些狀態沒變。而 require 函式應被用來作某個條件是否滿足，像是輸入或是狀態變數，或是用來來驗證其他函式的回傳值。如果使用得當，會有一些分析的工具可以來檢查什麼狀況下會進到 assert 。適當的程式碼不應該有進到 assert 的機會。如果進到assert 代表合約某個地方有問題，應該要找出來並修復。

另外還有兩個方式可以觸發異常，一為 revert 函式，另一為 throw 關鍵字。revert 函式可以用來標示錯誤和回復狀態。之後 revert 函式也可以加入該異常的描述。而 throw 關鍵字等同於 revert()

!Note: 0.4.13 版之後 throw 被棄用，也會在未來得版本直接捨棄。

當一個異常發生在子呼叫(sub-call)裡，會自動向上發出異常。除非這個異常是由 send 或是低階語言 call, delegatecall 和 callcode，當這些指令發生異常，會像上回傳 false 。

!Warning: 在 EVM 的設計裡，如果呼叫的帳號不存在，call, delegatecall 和 callcode 會回傳成功。 

捕獲異常是不可能的。

在接下來的例子裡可以看到 require 怎麼用來檢查條件，以及 assert 怎麼被用來作內部錯誤的檢查:

```
pragma solidity ^0.4.0;

contract Sharer {
    function sendHalf(address addr) public payable returns (uint balance) {
        require(msg.value % 2 == 0); // Only allow even numbers
        uint balanceBeforeTransfer = this.balance;
        addr.transfer(msg.value / 2);
        // Since transfer throws an exception on failure and
        // cannot call back here, there should be no way for us to
        // still have half of the money.
        assert(this.balance == balanceBeforeTransfer - msg.value / 2);
        return this.balance;
    }
}
```
以下列出會發出 assert 異常的情況：
1. 如果你用過大或是負的索引(index)來存取陣列 (即x[i] where i >= x.length)
2. 如果你用過大或是負的索引來存取一個固定長度的 bytesN
3. 如果你對0作除法或是取餘數 (像是: 5/0 或 23%0)
4. 如果用負數來做位移 (shift)
5. 如果用過大或是負的值來對應到enum類型的值。
6. 如果你在內部函式裡呼叫零初始化 (zero-initialized)的變數
7. 如果你用 assert 來檢查且內容結果為false。

以下列出會發出 require 異常的情況：
1. 呼叫 throw.
2. 如果你用 require 來檢查且內容結果為false。
3. 如果你用信息呼叫(message call)來呼叫函式，但這個函式沒有被正確的執行完成。(像是 gas 不夠用，沒有對應到的函式對象，或是該韓式自己丟出異常)。除了用call, send, delegatecall 或是 callcode 會回傳 false 而不會丟出異常。
4. 如果你用 new 關鍵字來新建合約，但沒有正確的新建完成。(例子請見前文有提到的"not finish properly"沒有適當完成)
5. 如果你對一個沒有程式碼的合約呼叫外部函式。
6. 如果合約透過沒有宣告 payable 的函式接受到 Ether。(包括建構子和 fallback 函式)
7. 如果合約透過一個公開的 getter 函式接收到 Ether
8. 如果 .transfer() 失敗。

對於 require 類型的異常，Solidity 用還原操作(0xfd 指令)來處理。而 assert 類型的異常用非法操作(invalid operation, 0xfe 指令)來處理。不管哪種異常，EVM都會把所有對合約狀態做的變動回復。而做回復的原因是因為沒有安全的方法可以繼續執行，且預期的結果沒有發生。由於我們想保留事務的原子性,（所以）最安全的做法是恢復所有的變動，並使整個事務(或者至少調用)沒有受影響。當 assert 類型的異常發生，會消耗掉所有能用的 gas。而在大都會版本後，require 類型的異常不會繼續消耗 gas。

© Copyright 2015, Ethereum. Revision 37381072. (尚未更新)

Built with [Sphinx](http://sphinx-doc.org/) using a [theme](https://github.com/snide/sphinx_rtd_theme) provided by [Read the Docs](https://readthedocs.org/).

 Read the Docsv: latest 
