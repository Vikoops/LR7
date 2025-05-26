## Лабораторная работа №7: Преобразование и анализ кода с использованием Clang и LLVM

**Тема:** Преобразование и анализ кода с использованием Clang и LLVM.

**Цель работы:** Познакомиться с инструментами Clang и LLVM, научиться собирать AST и IR-промежуточное представление кода на C/C++, а также извлекать базовую информацию о программе.

**В соответствии с вариантом задания необходимо:**
   1. Установить Clang и LLVM;
   2. Скомпилировать простой C-файл с использованием clang и получить его: абстрактное синтаксическое дерево (AST), промежуточное представление LLVM IR;
   3. Использовать opt для применения базовой комплексной оптимизации (например, О2);
   4. Построить граф потока управления (CFG) для оптимизированной программы;
   5. Проанализировать результат, сделать выводы и ответить на контрольные вопросы.

### Ход работы

**1. Установка и подготовка среды**

Работа выполнялась в среде Ubuntu 22.04.5 LTS. Установлены следующие инструменты:
- clang - компилятор языка C/C++;
- llvm - инструменты анализа и оптимизации кода;
- opt - инструмент для работы с LLVM IR и применения оптимизаций;
- Graphviz - инструмент для визуализации кода.

Команда установки: ```sudo apt install clang llvm```

![Установка инструментов](https://github.com/Vikoops/LR7/blob/main/image/install.png)

**2. Исходный код**

Программа на языке C:
```c
#include <stdio.h>

int square(int x) {
   return x * x;
}

int main() {
   int a = 5;
   int b = square(a);
   printf("%d\n", b);
   return 0;
}
```

Сохранена в файл main.c.

![Файл main.c](https://github.com/Vikoops/LR7/blob/main/image/mainc.png)

**3. Получение AST**

Команда: ```clang -Xclang -ast-dump -fsyntax-only main.c```

![Получение AST](https://github.com/Vikoops/LR7/blob/main/image/step3.png)

Функция square принята, содержит параметр x и возвращает x * x.

**4. Генерация LLVM IR**

Команда: ```clang -S -emit-llvm main.c -o main.ll```

![Генерация LLVM IR](https://github.com/Vikoops/LR7/blob/main/image/step4.png)

**5. Оптимизация IR**

Команда: ```clang -O0 -S -emit-llvm main.c -o main_O0.ll```

Стоит отметить, что в файле с IR (main.ll) до оптимизации:
   - Все переменные (a, b, x.addr) размещены в памяти через alloca;
   - Множество операций load и store;
   - square вызывается как отдельная функция.

![Файл main_O0.ll](https://github.com/Vikoops/LR7/blob/main/image/step5.png)

Команда: ```clang -O2 -S -emit-llvm main.c -o main_O2.ll```

Команда -O2 - комплексная оптимизация среднего уровня. Она применяет более 30 различных оптимизаций:
   - -inline - встраивание небольших функций (встраивает square в main, если она вызывается один раз);
   - -constprop - подставит значение square(5) → 25, если функция встроена и всё известно на этапе компиляции;
   - -mem2reg - перевод переменных из памяти в регистры (SSA);
   - -instcombine - объединение и упрощение инструкций (упростит арифметику, например, x * x может быть преобразовано в shl при x = 2^n);
   - -simplifycfg - оптимизирует структуру блоков (упростит граф управления, если после inlining останутся лишние блоки);
   - -reassociate, -gvn, -sroa, -dce и другие.

В файле с IR после оптимизации:
   - Вся функция square исчезла - она была встроена (-inline) и затем вычислена (оптимизация -constprop);
   - Никаких переменных, alloca, store, load - всё удалено (оптимизации -mem2reg, -dce);
   - Остался только вызов printf(25).

![Файл main_O2.ll](https://github.com/Vikoops/LR7/blob/main/image/step5_1.png)

Команда: ```diff main_O0.ll main_O2.ll```

Сравнение двух файлов:

![Сравнение двух файлов](https://github.com/Vikoops/LR7/blob/main/image/step5_2.png)

Стоит отметить, что после оптимизации произошли следующие изменения:
   - Переменные типа alloca были удалены;
   - Код переведён в SSA-форму;
   - Оптимизация улучшила читаемость и упростила поток управления.

**6. Граф потока управления программы**

Команда для генерации оптимизированного LLVM IR: ```clang -O2 -S -emit-llvm main.c -o main.ll```

Команда для генерации .dot-файлов CFG для функций: ```opt -dot-cfg -disable-output main.ll```

![Генерация .dot-файлов](https://github.com/Vikoops/LR7/blob/main/image/step6.png)

Эта команда создаст DOT-файлы:
   - .main.dot - для функции main;
   - .square.dot - для square, если она не была удалена оптимизацией.

![.dot-файлы](https://github.com/Vikoops/LR7/blob/main/image/step6_1.png)

Команда для установки библиотеки Graphviz: ```sudo apt install graphviz```

![Установка библиотеки Graphviz](https://github.com/Vikoops/LR7/blob/main/image/step6_2.png)

Команды для преобразования файлов с расширением .dot в .png с помощью Graphviz:
   - ```dot -Tpng .main.dot -o cfg_main.png```
   - ```dot -Tpng .square.dot -o cfg_square.png```

![Преобразования файлов .dot в .png](https://github.com/Vikoops/LR7/blob/main/image/step6_3.png)
![](https://github.com/Vikoops/LR7/blob/main/image/step6_4.png)

Команды для просмотра файлов с CGF:
   - ```xdg-open cfg_main.png```

![Просмотр файла cfg_main.png](https://github.com/Vikoops/LR7/blob/main/image/step6_5.png)

   - ```xdg-open cfg_square.png```

![Просмотр файла cfg_square.png](https://github.com/Vikoops/LR7/blob/main/image/step6_6.png)

Стоит отметить, что в LLVM каждый граф потока управления (CFG) строится на уровне функции, поскольку структура управления всегда локальна для тела функции. Для получения полного представления о программе, нужно построить CFG для всех функций и анализировать их совокупность. Автоматическое объединение всех CFG в один граф не предусмотрено в LLVM по умолчанию.

### Команды терминала

```bash
sudo apt install clang llvm
clang -Xclang -ast-dump -fsyntax-only main.c
clang -S -emit-llvm main.c -o main.ll
clang -O0 -S -emit-llvm main.c -o main_O0.ll
clang -O2 -S -emit-llvm main.c -o main_O2.ll
diff main_O0.ll main_O2.ll
clang -O2 -S -emit-llvm main.c -o main.ll
opt -dot-cfg -disable-output main.ll
sudo apt install graphviz
dot -Tpng .main.dot -o cfg_main.png
dot -Tpng .square.dot -o cfg_square.png
xdg-open cfg_main.png
xdg-open cfg_square.png
```

### Промежуточные выводы по каждому заданию

**1. Установка и подготовка среды**

Установлены необходимые инструменты на Ubuntu 22.04.5 LTS: ```clang```, ```llvm```, ```opt```, ```graphviz```.
Среда полностью готова к анализу C-кода и визуализации. Инструменты работают корректно.

**2. Исходный код**

Создан файл ```main.c``` с простой функцией ```square(x)``` и вызовом ```printf```.
Код компилируется без ошибок и подходит для анализа.

**3. Получение AST**

С помощью команды ```clang -Xclang -ast-dump -fsyntax-only main.c``` успешно построено дерево синтаксического разбора (AST).
Функции и структуры распознаны корректно, дерево построено.

**4. Генерация LLVM IR**

Получено промежуточное представление (IR) в текстовом формате ```.ll```. Код транслируется в LLVM IR без ошибок.

**5. Оптимизация IR**

Сравнение IR-файлов до (```-O0```) и после (```-O2```) оптимизации показывает:
   - Удаление временных переменных (```alloca```, ```store```, ```load```);
   - Встраивание функции ```square``` (inlining);
   - Замена вычислений на константы (```constprop```);
   - Применение SSA-формы (```mem2reg```) и других оптимизаций.

Оптимизации LLVM работают эффективно, уменьшая и упрощая код.

**6. Граф потока управления программы**

Сгенерированы ```.dot```-файлы для каждой функции, а затем преобразованы в ```.png``` с помощью ```Graphviz```.
   - ```main.dot``` успешно визуализирован;
   - ```square.dot``` присутствует только если не удалена оптимизацией.

CFG успешно построены, визуализация отражает структуру управления в функциях.

### Выводы

- С помощью Clang можно получить полную структуру AST и IR, а также CGF;
- LLVM предоставляет гибкие инструменты анализа и оптимизации;
- Промежуточное представление кода удобно для написания компиляторных трансформаций.
