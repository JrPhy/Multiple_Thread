與記憶體池一樣，我們不希望在執行期間一直創建與銷毀線程，此舉會增加開銷，所以也是在一開始就開好許多線程等待使用。當然任務也有可能是比線程數還要多，所以就會需要把任務放進佇列中，當然佇列也不可能是無限大，當佇列滿時就無法放入佇列中。另外還需要條件變數跟鎖，當加入任務時必須先要看 pool 是否還有空間，否則無法將任務加入。

## 1.資料結構
```C
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

typedef struct {
    void (*function)(void *);
    void *arg;
} Task;

typedef struct {
    Task *task_queue;
    int head, tail, count, queue_size; // 環狀佇列
    int thread_count; // 線程數量
    int stop; // 停止 flag
    pthread_mutex_t lock;
    pthread_cond_t cond;
    pthread_t *threads;
} ThreadPool;
```
Task 中的兩個成員是給 pthread 使用，ThreadPool 中則是包含鎖與環狀佇列等變數
## 2. 初始化
```C
void *worker_thread(void *arg) {
    ThreadPool *pool = (ThreadPool *)arg;
    while (1) {
        pthread_mutex_lock(&pool->lock);
        while (pool->count == 0 && !pool->stop) {
            pthread_cond_wait(&pool->cond, &pool->lock);
        }

        if (pool->stop && pool->count == 0) {
            pthread_mutex_unlock(&pool->lock);
            pthread_exit(NULL);
        }

        Task task = pool->task_queue[pool->head];
        pool->head = (pool->head + 1) % pool->queue_size;
        pool->count--;
        pthread_mutex_unlock(&pool->lock);

        task.function(task.arg);
    }
}

void thread_pool_init(ThreadPool *pool, int thread_count, int queue_size) {
    // 初始化環狀佇列
    pool->head = pool->tail = pool->count = 0;
    pool->queue_size = queue_size;
    pool->thread_count = thread_count;
    pool->stop = 0;
    
    pool->task_queue = malloc(sizeof(Task) * queue_size);
    pool->threads = malloc(sizeof(pthread_t) * thread_count);

    pthread_mutex_init(&pool->lock, NULL);
    pthread_cond_init(&pool->cond, NULL);

    for (int i = 0; i < thread_count; i++) {
        pthread_create(&pool->threads[i], NULL, worker_thread, pool);
    }
}
```
worker_thread 就是我們開啟的線程，因為 pthread_create 的關係，所以需要先將實作寫好，初始化時就要先傳入。worker_thread 中可以看到當沒有可用 thread 時就需要等待，直到被通知有空的線程時才會繼續往下，反之如果有空的線程，就將其佇列中等待的線程拿出來執行。其中使用條件變數而不是一直 busy wait，可以減少等待時的開銷。
## 2. 加入與執行任務
```C
void execute_task(void *arg) {
    int id = *(int *)arg;
    printf("任務 %d 開始執行\n", id);
    sleep(1);
    printf("任務 %d 完成\n", id);
    free(arg);
}

void thread_pool_add_task(ThreadPool *pool, void (*function)(void *), void *arg) {
    pthread_mutex_lock(&pool->lock);
    int next_tail = (pool->tail + 1) % pool->queue_size;
    if (pool->count < pool->queue_size) {
        pool->task_queue[pool->tail] = (Task){function, arg};
        pool->tail = next_tail;
        pool->count++;
        pthread_cond_signal(&pool->cond);
    }
    pthread_mutex_unlock(&pool->lock);
}
```
再加入任務時要把環狀佇列保護起來，否則可能會有 Race condition
```C
void thread_pool_shutdown(ThreadPool *pool) {
    pthread_mutex_lock(&pool->lock);
    pool->stop = 1;
    pthread_cond_broadcast(&pool->cond);
    pthread_mutex_unlock(&pool->lock);

    for (int i = 0; i < pool->thread_count; i++) {
        pthread_join(pool->threads[i], NULL);
    }

    free(pool->task_queue);
    free(pool->threads);

    pthread_mutex_destroy(&pool->lock);
    pthread_cond_destroy(&pool->cond);
}
```
```C
int main(int argc, char *argv[]) {
    int thread_count = 4;
    int queue_size = 10;

    ThreadPool pool;
    thread_pool_init(&pool, thread_count, queue_size);

    for (int i = 0; i < 10; i++) {
        int *task_id = malloc(sizeof(int));
        *task_id = i;
        thread_pool_add_task(&pool, execute_task, task_id);
    }

    sleep(5);  // 讓線程執行任務
    thread_pool_shutdown(&pool);
    return 0;
}
```
