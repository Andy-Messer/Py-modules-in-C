# PyTrace module in C

---
## PyTrace - и с чем его едят.

> Сallback that the Python interpreter will invoke for every function call and line of Python executed.

>  ЗАПОМНИТЕ ЭТО!!! Функция trace срабатывает на каждой строке, на каждом вызове, каждом исключении.

 - простейший пример trace функции.
```python
import sys

def my_py_trace_fn(frame, event, arg):
    # .. do something useful ..
    return my_py_trace_fn

sys.settrace(my_py_trace_fn)
```
 1) frame - текущий исполняемый frame.
 2) event - строка с информацией о типе event-а.
 3) arg - аналог returnable value, когда мы хотим вернуть код возврата или ошибки.

**Важно!** Надо понимать, что каждый раз мы возвращаем саму функцию. Это важно для понимания вопроса: **"Что такое цепочка трассировки?"**

---
## 20к лье под водой - спускаемся к C.

 На дне Python, средства трассировки реализованны с помощью C. Каждый поток исполнения имеет два глобальных указателя:
 1) c_tracefunc - указывает на сигнатуру вызванной функции.
```c
int my_trace(PyObject *obj, PyFrameObject *frame, int event, PyObject *arg);
```
 2) c_traceobj - указывает на произвольный объект Python, который будет передан в качестве первого аргумента функции.
 3) Остальные аргументы аналогичны аргументам функции Python, за исключением того, что здесь событие — это целое число, а не строка.

Для регистрации trace_func, используем [PyEval_SetTrace()](https://docs.python.org/3/c-api/init.html?highlight=pyeval_settr#c.PyEval_SetTrace).

> P.s. К сожалению, в C **НЕТ** такого же простого механизма цепочек, как упоминалось ранее.

Достаточно естественный факт, что sys.settrace() в Python реализована с помощью PyEval_SetTrace().

Простейший пример обертки над PyEval_SetTrace():
```c
PyObject* sys_settrace(PyObject *py_fn)
{
    if (py_fn == Py_None) {
        PyEval_SetTrace(NULL, NULL);
    }
    else {
        PyEval_SetTrace(trace_trampoline, py_fn);
    }
    return Py_None;
}
```
Рассмотрим код выше, если мы передадим в sys_settrace() NULL, то функция трассировки будет затёрта, иначе же мы можем записать функцию trace_trampoline.

Пример trace_trampoline:
```c++
int
trace_trampoline(
    PyObject *obj, PyFrameObject *frame, int event, PyObject *arg
)
{
    PyObject *callback;

    if (event == PyTrace_CALL) {
        /* Remember obj is really my_py_trace_fn */
        callback = obj;
    }
    else {
        callback = frame->f_trace;
    }
    if (callback == NULL) {
        return 0;
    }
    result = /* Call callback(frame, event as str, arg) */;
    if (/* error occurred in callback */) {
        PyEval_SetTrace(NULL, NULL);
        frame->f_trace = NULL;
        return -1;
    }
    if (result != Py_None) {
        frame->f_trace = result;
    }
    return 0;
}
```
В коде выше происходит интересная ситуация, мы проэмулировали цепочку трассировки, через frame->f_trace.

### Баги и приколы

 1) Кроме метода sys.Settrace(), есть ещё sys.gettrace(). Прикол:

```python
# import sys
sys.settrace(sys.gettrace())
```
На Python данный код отработает корректно, однако на C, будет тяжело вылечить свою прогу после такого))))

---
> Здесь представлена далеко не вся информация из статьи, если кто-то хочет ещё изучать то вот:
> 1) [статья](https://nedbatchelder.com/text/trace-function.html)
> 2) [Python/C API Reference Manual](https://docs.python.org/3/c-api/index.html)
> 3) [Extending and Embedding the Python Interpreter](https://docs.python.org/3/extending/index.html)
> 4) [Frame Objects](https://docs.python.org/3.12/c-api/frame.html)
> 5) [Русская документация](https://digitology.tech/docs/python_3/c-api/init.html#:~:text=Py_tracefunc)
> 6) [Статья с курса по АКОСу](https://github.com/victor-yacovlev/mipt-diht-caos/tree/master/practice/python)
