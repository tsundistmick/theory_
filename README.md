# Синтаксис многопоточности в C++ (C++11 и новее)

Полная шпаргалка по основным конструкциям: потоки, мьютексы, блокировки и синхронизация.

---

## 1. Заголовочные файлы

```cpp
#include <thread>     // std::thread, std::this_thread
#include <mutex>      // std::mutex, std::lock_guard, std::unique_lock и др.
#include <shared_mutex> // C++17: std::shared_mutex, std::shared_lock
#include <chrono>     // std::chrono::milliseconds и таймауты
#include <future>     // std::async, std::future, std::promise
#include <atomic>     // std::atomic (атомарные операции)
#include <condition_variable> // std::condition_variable

2. Потоки (std::thread)
Создание потока
cpp

// Функция
void foo(int a, double b) { /*...*/ }

// Лямбда
auto bar = [](std::string s) { /*...*/ };

// Запуск
std::thread t1(foo, 42, 3.14);      // аргументы после функции
std::thread t2(bar, "hello");
std::thread t3([](){ /*...*/ });    // лямбда без аргументов

Методы std::thread
cpp

t.join();              // Ожидать завершения потока
t.detach();            // Отсоединить поток (будет жить сам)
t.joinable();          // Можно ли вызвать join()?
std::this_thread::sleep_for(std::chrono::milliseconds(100)); // усыпить текущий поток
std::this_thread::get_id(); // ID текущего потока

3. Мьютексы (std::mutex и семейство)
Типы мьютексов
cpp

std::mutex m;              // Простой нерекурсивный мьютекс
std::recursive_mutex rm;   // Рекурсивный (можно захватывать из того же потока)
std::timed_mutex tm;       // Поддерживает try_lock_for / try_lock_until
std::shared_mutex sm;      // C++17: разделяемый (для read/write блокировок)

Ручное управление (не рекомендуется)
cpp

m.lock();      // Захватить (ждет, если занято)
m.unlock();    // Освободить
m.try_lock();  // Попытаться захватить (возвращает bool)

4. Автоматические блокировки (RAII)
std::lock_guard (самый простой)
cpp

{
    std::lock_guard<std::mutex> lock(m);  // lock() при создании
    // защищенная секция
} // автоматический unlock() при разрушении

std::unique_lock (гибкий)
cpp

std::unique_lock<std::mutex> lock(m);                // сразу lock
std::unique_lock<std::mutex> lock(m, std::defer_lock); // не lock сразу
std::unique_lock<std::mutex> lock(m, std::try_to_lock); // попытаться lock

lock.lock();     // ручной lock
lock.unlock();   // ручной unlock (и снова lock можно)
lock.try_lock(); // попытаться захватить

std::shared_lock (C++17, для shared_mutex)
cpp

std::shared_mutex rw_mutex;

// Чтение (можно нескольким потокам одновременно)
{
    std::shared_lock lock(rw_mutex);
    // читаем данные
}

// Запись (только одному)
{
    std::unique_lock lock(rw_mutex); // или lock_guard
    // пишем данные
}

Блокировка нескольких мьютексов сразу (без deadlock-ов)
cpp

std::lock(m1, m2);  // блокирует оба атомарно

// В паре с unique_lock defer_lock:
std::unique_lock<std::mutex> lock1(m1, std::defer_lock);
std::unique_lock<std::mutex> lock2(m2, std::defer_lock);
std::lock(lock1, lock2); // блокируем оба через локеры

5. Таймауты (для timed_mutex)
cpp

std::timed_mutex tm;

// Ждать не больше 100 мс
if (tm.try_lock_for(std::chrono::milliseconds(100))) {
    // захватили
    tm.unlock();
}

// Ждать до определенного момента времени
auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(1);
if (tm.try_lock_until(deadline)) {
    // захватили
    tm.unlock();
}

6. Однократная инициализация
cpp

std::once_flag flag;

void init() { /* выполнится только один раз */ }

std::call_once(flag, init);

7. Условные переменные (condition_variable)
cpp

std::mutex m;
std::condition_variable cv;
bool ready = false;

// Поток-производитель
{
    std::lock_guard<std::mutex> lock(m);
    ready = true;
    cv.notify_one(); // или notify_all()
}

// Поток-потребитель
{
    std::unique_lock<std::mutex> lock(m);
    cv.wait(lock, []{ return ready; }); // ждем с предикатом
    // ready == true, работаем
}

8. Высокоуровневый синтаксис (std::async)

Самый простой способ запустить асинхронную задачу и получить результат.
cpp

// Запуск в отдельном потоке (или отложенно)
std::future<int> result = std::async(std::launch::async, [](){
    return 42;
});

// Получить результат (блокирует, если еще не готов)
int value = result.get();

// Политики запуска:
// std::launch::async     - сразу в новом потоке
// std::launch::deferred  - лениво, при вызове get()

9. Атомарные операции (std::atomic)

Для простых типов без мьютексов (быстрее).
cpp

std::atomic<int> counter(0);

counter++;                     // атомарно
int x = counter.load();        // атомарно прочитать
counter.store(10);             // атомарно записать
int old = counter.exchange(5); // атомарно заменить и вернуть старое

10. Короткие правила выбора
Задача	Инструмент
Просто защитить участок кода	std::lock_guard<std::mutex>
Нужен ручной unlock или таймаут	std::unique_lock<std::mutex>
Много читателей, редкие записи	std::shared_lock + std::shared_mutex
Нужно заблокировать 2+ мьютекса	std::lock() + std::unique_lock с defer_lock
Простая переменная (флаг, счетчик)	std::atomic<T>
Разбудить другой поток по событию	std::condition_variable
Получить результат из потока	std::async + std::future
11. Типичная структура защищенного кода
cpp

#include <iostream>
#include <thread>
#include <mutex>
#include <vector>

std::mutex mtx;
int shared_data = 0;

void worker(int id) {
    for (int i = 0; i < 100; ++i) {
        // Вход в критическую секцию
        std::lock_guard<std::mutex> lock(mtx);
        shared_data++;  // работа с общими данными
        // Выход - автоматически при разрушении lock_guard
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i)
        threads.emplace_back(worker, i);
    
    for (auto& t : threads)
        t.join();
    
    std::cout << shared_data << std::endl; // 1000
    return 0;
}
