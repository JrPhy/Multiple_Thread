再多執行緒或多進程的程式中，用鎖來保護臨界區域是最常見的做法，但是跟原子操作效能比起來也差很多，最有效率的寫法就是既可以多執行緒，又可以不用***上鎖***或是不會***忙等待***，使得整個程式的併行或併發度更高。雖然程式沒有鎖，但有可能陷入無窮迴圈使得看起來像被鎖住一樣，例如
```C++
while (x == 0) {
    x = 1 - x;
}
```
當有兩個執行緒 A, B 同時執行上面這段程式碼時，有可能

|x |Thread A | Thread B |
|:--|:--|:--|
|0 | while (x == 0)	 |  | 
|0 |  | while (x == 0) | 
|1 | x = 1 - x	 |  | 
|0 |  | x = 1 - x | 
|0 | while (x == 0)	 |  | 
|0 |  | while (x == 0) | 
|0 | ... | ... | 

雖然沒有上鎖，但是因為 Race condition 的關係使得無法跳脫迴圈，所以這也不算 lock-free 的程式。除了無鎖外，執行緒安全也是必要的，也就是 A 在讀取的時候並不能由其他執行緒修改。一種避免的方法就是用 CAS，當然 CAS 也會有 ABA 問題，就可以再加上版本號來解決。

## 一、LOCK-FREE STACK
```c++
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

struct Node {
    int value;
    Node* next;
    Node(int v) : value(v), next(nullptr) {}
};

struct TaggedPtr {
    Node* ptr;
    unsigned tag;
};

std::atomic<TaggedPtr> head;

void push(int v) {
    Node* newNode = new Node(v);
    TaggedPtr oldHead, newHead;
    do {
        oldHead = head.load(std::memory_order_relaxed);
        newNode->next = oldHead.ptr;
        newHead = { newNode, oldHead.tag + 1 }; // tag +1
    } while (!head.compare_exchange_weak(oldHead, newHead));
}

Node* pop() {
    TaggedPtr oldHead, newHead;
    do {
        oldHead = head.load();
        if (!oldHead.ptr) return nullptr;
        newHead = { oldHead.ptr->next, oldHead.tag + 1 };
    } while (!head.compare_exchange_weak(oldHead, newHead));
    return oldHead.ptr;
}

int main() {
    head.store({nullptr, 0});

    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) threads.emplace_back(push, i);
    for (auto& t : threads) t.join();

    while (Node* n = pop()) {
        std::cout << "Popped " << n->value << "\n";
        delete n;
    }
}
```
在 Node 的結構中多加了一個 tag，每次操作時都要 tag+1 然後去做 CAS，這樣就可以做到多執行緒的 lock-free。不過在此並沒有刪除所開出來的指標，因為再多執行緒的狀態下，其中一個執行緒刪除了該指標有可能另一個還在讀取，就會造成程式崩壞，在此可以使用 Hazrad pointer 來解決。

## 二、Hazrad pointer
每個執行緒在存取節點時會先登記 hazard pointer，表示「我正在用這個指標」。回收時必須檢查 hazard list，如果該節點仍在 hazard list，就不能釋放。所以 pop 只能「取出節點」，不能馬上 delete，也就是要延遲回收。
```c++
#include <atomic>
#include <vector>
#include <thread>
#include <iostream>

struct Node {
    int value;
    Node* next;
    Node(int v) : value(v), next(nullptr) {}
};

struct TaggedPtr {
    Node* ptr;
    unsigned tag;
};

std::atomic<TaggedPtr> head{TaggedPtr{nullptr,0}};

constexpr int MAX_THREADS = 8;
std::atomic<Node*> hazardPointers[MAX_THREADS];

thread_local std::vector<Node*> retireList;

void setHazard(int tid, Node* ptr) {
    hazardPointers[tid].store(ptr, std::memory_order_release);
}
void clearHazard(int tid) {
    hazardPointers[tid].store(nullptr, std::memory_order_release);
}
bool isHazard(Node* ptr) {
    for (int i=0; i<MAX_THREADS; i++) {
        if (hazardPointers[i].load(std::memory_order_acquire) == ptr) return true;
    }
    return false;
}

void safeDelete(Node* node) {
    retireList.push_back(node);
    // 嘗試回收
    for (auto it = retireList.begin(); it != retireList.end();) {
        if (!isHazard(*it)) {
            delete *it;
            it = retireList.erase(it);
        } else ++it;
    }
}

void push(int v) {
    Node* newNode = new Node(v);
    TaggedPtr oldHead, newHead;
    do {
        oldHead = head.load();
        newNode->next = oldHead.ptr;
        newHead = {newNode, oldHead.tag+1};
    } while (!head.compare_exchange_weak(oldHead, newHead));
}

Node* pop(int tid) {
    TaggedPtr oldHead, newHead;
    do {
        oldHead = head.load();
        if (!oldHead.ptr) return nullptr;
        setHazard(tid, oldHead.ptr);
        if (oldHead.ptr != head.load().ptr) continue;
        newHead = {oldHead.ptr->next, oldHead.tag+1};
    } while (!head.compare_exchange_weak(oldHead, newHead));
    clearHazard(tid);

    safeDelete(oldHead.ptr); // 延遲回收
    return oldHead.ptr;
}

int main() {
    for (int i=0; i<MAX_THREADS; i++) hazardPointers[i].store(nullptr);

    for (int i=0; i<5; i++) push(i);

    std::vector<std::thread> threads;
    for (int i=0; i<3; i++) {
        threads.emplace_back([i] {
            while (pop(i)) {}
        });
    }
    for (auto& t : threads) t.join();
}
```
在 pop 時就先丟進 hazard 中去管理，最後再檢查是否能移除，就可以達到多執行緒下安全移除指標了。
