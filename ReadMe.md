程式 Program, 進程 Process 與線程(執行緒) Thread 的差異：
Program：一段程式碼寫好但尚未被執行 \
Process：程式被執行後會佔有**資源**，並且由 OS 去分配資源。每個進程之間是互相獨立的，可以藉由訊息傳遞(Message passing)或記憶體共享(Share Memory)來交換資料，達到多進程的效果。一個程式會佔據一個 CPU 的核心，多核心的 CPU 就可以同時執行多個程式。而每個程式裡面還有一個或多個線程，可以看成是線程的容器。每個程式在 OS 中會被賦予一個 pid，在 windows 中可以用工作管理員看，linux 則是用 top 查看。\
Thread：Process 分配到資源後，實際執行者是線程，是 OS 分配 CPU-time 的單位，也就是執行的最小單位。每個程式內的執行緒可以直接共享全域變數。

## 1. Concurrency 並行 和 Parallelism 平行
並行與平行的差異在於是由幾個 CPU 核心在處理，若有多個任務在**多個 CPU 核心**處理則稱為平行，而多個任務在**一個 CPU 核心**處理則稱為併行。
#### 1. Parallelism 平行
在 C/C++ 中，平行的任務會有多個 main() 函數同時被執行，例如在算矩陣的 det 時，我可以整個矩陣一起算，也可以把矩陣分成多個小方陣，算出個別的 det 在相乘。\
det(A) = det(A<sub>1</sub>)det(A<sub>2</sub>)...det(A<sub>n</sub>)\
這時就可以寫一個計算 det 的 main 函數，然後分別把 A<sub>1</sub>、A<sub>2</sub> ... A<sub>n</sub> 的資料丟進作計算，最後算出來在做相乘。而並行則是把 A 中的所有元素傳進一個 main() 裡面，再去開 thread 去分別做計算，算完後直接在 main() 裡面做相乘，結束後算出來的就是 det(A)。例如有個程式 det_cal 如下
```cpp
#include <iostream>
#include <fstream>
#include <cstdlib>    //使用exit必須include
using namespace std;
int main(int argc, char *argv[]){
    ifstream in;
    in.open(argv[1]);
    // 計算行列式值
    return 0;
}
```
另外有個陣列元素存於 A.txt 中，若要分成四個矩陣去做運算，那需要自行將檔案拆成四分，然後個別執行，最後再將計算的值乘起來即為 det(A)。如果為 linux，則需要開四個 terminal 然後個別去跑
```
./det_cal A1.txt
./det_cal A2.txt
./det_cal A3.txt
./det_cal A4.txt
```
#### 2. Concurrency 並行
