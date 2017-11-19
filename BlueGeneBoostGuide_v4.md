# BlueGeneBoostGuide v4

1. Собрать generator.cpp на BlueGene можно следующей командой:
  ```bash
  $ xlc++_r -qsmp=omp generator.cpp -o generator -O3
  ```

2. **ВАЖНО: ** в большинстве компиляторов С/С++ важен порядок линкуемых бинарей. В старых версиях мануала был указан неправильный порядок: сначала библиотеки (`-lboost_lib …`), а потом `main.cpp`, а теперь наоборот, т.е. сначала ваша программа, а потом библиотеки.
  Собирать свой файл можно следующей командой, где `main.cpp` — ваш исходник. На с++11 даже не надейтесь, его нет. Этот буст собрал какой-то студент 615 группы прошлого года. Спасибо ему.

  ```bash
  $ mpicxx -O3 -I /gpfs/data/edu-cmc-stud16-615-09/dev/libs/libboost/include -L /gpfs/data/edu-cmc-stud16-615-09/dev/libs/libboost/lib main.cpp -o main -lboost_graph_parallel -lboost_mpi -lboost_serialization -lboost_system
  ```

  Для тех, кто не очень понимает, что тут происходит: мы указываем компилятору путь до библиотек (`-L`) и заголовочных файлов (`-I`). Дальше говорим ему, какие библиотеки линковать (`-llibrary_name`). Возможно, вам пригодится слинковать ещё какие-то библиотеки.

  P.S. В начале файла `main.cpp` должна быть строка `#include <boost/graph/use_mpi.hpp>` (можете в моей домашней папке глянуть пример). Это нужно, чтобы буст работал с mpi.

3. Запускать командой:

   ```bash
   $ mpisubmit.bg -n 128 -w 00:15:00 main
   ```

   В той же папке вскоре появятся файлы `main.*.err`, `main.*.out`. В них будут ошибки и вывод ваших программ. Для вас число программ, время счёта и имя запускаемого файла могут быть другими (тут это 128, 15 минут и `main` соответственно).

# Всякие полезности

## Просмотр очереди

```bash
# Только мои задачи
$ llq -u <username>
# все задачи
$ llq
```

## Отмена задачи

```bash
# Отмена по id-задачи. Для примера id равен fen1.10985723.0
$ llcancel fen1.10985723.0
```

## Команда watch

```bash
# Просто полезная команда, которая запускает для вас периодически что-то
# Например, ls
# Так вы будете видеть нужные изменения на экране, не набирая команду ls постоянно
$ watch ls
Every 2.0s: ls

clean.sh                                            
generator                                           
generator.cpp                                       
graph.data                                          
main                                                
main.10986198.out                                   
main.cpp                                            
make.sh                                             
run.sh   

# Когда задача появится / уйдет из очереди
$ watch llq
Every 2.0s: llq

Id                       Owner      Submitted   ST PRI Class        Running On                           
------------------------ ---------- ----------- -- --- ------------ -----------                          
fen1.10985723.0          mmg        11/8  03:06 R  50  n512_h24     fen1                                 
fen1.10986274.0          edu-cmc-sk 11/8  22:31 R  50  n256_m10     fen1                                 
fen1.10984639.0          zhukov_ka  10/17 23:01 I  50  n2048_m30    (alloc)                              
fen1.10984685.0          nordmike   10/28 21:15 I  50  n2048_m03                                         
fen1.10986275.0          edu-cmc-sk 11/8  22:32 I  50  n512_m05                                          

5 job step(s) in queue, 3 waiting, 0 pending, 2 running, 0 held, 0 preempted          
```

# Troubleshooting

## У меня не комплится с ошибкой `error: ‘some_shit’ is not a member of ‘another_shit’`

Наверняка это косяк компилятора (ошибка имеет место, например, в некоторых версиях `boost/graph/distributed/strong_components.hpp`), который не может использовать имя до его определения из-за вложенных namespace'ов. Например:

```cpp
namespace pupa {
  namespace lupa {
    int get_zarplata() {
      return pupa::get_zarplata();
    }
  }
  
  int get_zarplata() {
    return 0;
  }
}
```

Может помочь объявить прототипы нужных имён в начале заголовочного файла либо перенести описани повыше.

```cpp
int pupa::get_zarplata();

namespace pupa {
  namespace lupa {
    int get_zarplata() {
      return pupa::get_zarplata();
    }
  }
  
  int get_zarplata() {
    return 0;
  }
}
```

Или

```cpp
namespace pupa {
  int get_zarplata() {
    return 0;
  }
  
  namespace lupa {
    int get_zarplata() {
      return pupa::get_zarplata();
    }
  }
}
```

Пробуйте, экспериментируйте. Точного рецепта нет.

## Я очень хочу собрать boost сам

Могу только посочувствовать. В народе ходит мануал. Я его не проверял:

Ссылка на буст: http://www.boost.org/users/history/version_1_47_0.html

1. Скачайте архив на BlueGene

   ```bash
   wget --no-check-certificate -c 'http://sourceforge.net/projects/boost/files/boost/1.47.0/boost_1_47_0.tar/download'
   ```

2. Распакуйте


```bash
tar -xf boost147_0.tar.gz
```

3. Запустите `bootstrap.sh`

```bash
./bootstrap.sh --prefix="../boost_install" --with-toolset=gcc --with-libraries=mpi,filesystem,system,serialization,graph,graph_parallel
```

4. Откройте `<boost_root>/tools/build/v2/user-config.jam и добавьте строки:

```cpp
using gcc : 4.1.2 : mpicxx : ;
using mpi : /bgsys/drivers/ppcfloor/comm/bin/mpicxx ;
```

5. Соберите и установите

```bash
./b2 --prefix="../boost_install" --layout=versioned toolset=gcc variant=release install
```

## У меня программа ловит sigsegv (signal 11) при считывании графа

Используется граф, сгенерированный не на BlueGene. Сгенерируй его на BlueGene (см. выше). Или ты просто не умеешь работать с памятью в плюсах =)

## Мне нужно, чтобы один процесс считал граф, а потом передал другим по MPI

Используй ~~силу, Люк~~ `synchronize()`:

```cpp
synchronize(g.process_group());
```

P.S. Возможно, это не так. Однако так, без особых объяснений, записано в документации [здесь](http://www.boost.org/doc/libs/1_41_0/libs/graph_parallel/doc/html/distributed_adjacency_list.html) и [здесь](http://www.boost.org/doc/libs/1_51_0/libs/graph_parallel/doc/html/distributed_property_map.html).

## Не работает auto, а я не Бьорн Страуструп, чтобы писать типы переменных на восемь строчек!

1. Используйте auto.
2. Попробуйте собрать.
3. В полученном сообщении об ошибке приведения типов заберите название нужного вам (компилятор-то знает тип возвращаемого значения).

Пример:

```cpp
...
65: auto vla = make_vertex_list_adaptor(g);
...

...
main.cpp: In function ‘void parse(const std::string&)’:
main.cpp:65: error: ISO C++ forbids declaration of ‘vla’ with no type
main.cpp:65: error: cannot convert ‘boost::graph::vertex_list_adaptor<boost::adjacency_list<boost::vecS, boost::distributedS<boost::graph::distributed::mpi_process_group, boost::vecS, boost::defaultS>, boost::undirectedS, boost::no_property, boost::property<boost::edge_weight_t, float, boost::no_property>, boost::no_property, boost::listS>, boost::graph::stored_global_index_map<boost::vector_property_map<unsigned int, boost::local_property_map<boost::graph::distributed::mpi_process_group, boost::detail::parallel::global_descriptor_property_map<unsigned int>, boost::vec_adj_list_vertex_id_map<boost::no_property, unsigned int> > > > >’ to ‘int’ in initialization
...
```

В данном случае `boost::graph::vertex_list_adaptor<boost::adjacency_list<boost::vecS, boost::distributedS<boost::graph::distributed::mpi_process_group, boost::vecS, boost::defaultS>, boost::undirectedS, boost::no_property, boost::property<boost::edge_weight_t, float, boost::no_property>, boost::no_property, boost::listS>, boost::graph::stored_global_index_map<boost::vector_property_map<unsigned int, boost::local_property_map<boost::graph::distributed::mpi_process_group, boost::detail::parallel::global_descriptor_property_map<unsigned int>, boost::vec_adj_list_vertex_id_map<boost::no_property, unsigned int> > > > >` нужный вам тип. 

Любите плюсы, да? :3

## Я не понимаю, как графы работают в Boost. Многовато шаблонов, можно примеры?

```cpp
typedef adjacency_list<vecS,
        distributedS<mpi_process_group, vecS>, // распределённый
        undirectedS, // неориентированный
        no_property, // свойство вершины. Если не хотим свойства, пишем no_property
        property<edge_weight_t, float> // свойство ребра. В моём варианте вес
> ShittyGraph;
```

Ещё тут полезно и подробно про графы:

http://www.boost.org/doc/libs/1_65_1/libs/graph_parallel/doc/html/distributed_adjacency_list.html

Также полезно заглянуть в тесты в репозитории буста. Там всегда есть примеры использования функций. Пример для `dense_boruvka_minimum_spanning_tree`:

https://github.com/boostorg/graph_parallel/blob/4a62e6a7e17efa16ebca25717e48adc060b81641/test/distributed_mst_test.cpp

## Компилятор ругается на fstream(file_name, ios::in | ios::binary)

Попробуй заменить на `fstream file(file_name.c_str(), ios::in | ios::binary);`. Компилятор на BlueGene не может (да и зачем?) привести плюсовый `string` к `char *`.

## Мой алгоритм отработал, хочу вывести рёбра, но в ответе слишком много нулевых вершин

Надо было читать, как работает [распределённая хеш-таблица](http://www.boost.org/doc/libs/1_51_0/libs/graph_parallel/doc/html/distributed_property_map.html). Если пытаться вывести рёбра распределенного графа просто командами `source(edge, g); target(edge, g)`, то рискуете получить значение по умолчанию, а не реальное. Это связано с тем, что текущий процесс не хранит все рёбра, а запрашивать их у процессов-владельцев неявно не собирается.

Если не лезть в дебри синхронизации, то запросить правильное глобальное значение можно так:

```cpp
typedef graph_traits<YourGraph>::vertex_descriptor vertex_descriptor;
// u, v - дескрипторы вершин графа
vertex_descriptor u = source(edge, g);
vertex_descriptor v = target(edge, g);
// запрашиваем реальные значения у их владельцев
cout << g.distribution().global(owner(u), local(u)) << ", " << g.distribution().global(owner(v), local(v)) << endl;
```

Взято [отсюда](https://github.com/rlichtenwalter/radiant_model/blob/master/lib/boost_1_53_0/libs/graph_parallel/test/distributed_mst_test.cpp#L44).

## У меня не работает на графах больше 2^18^ вершин.

Попробуйте поставить время счёта побольше, например выставить параметр `-w 00:15:00`. Мне помогло (у меня вариант MST с алгоритмом Борувки).

Вообще на каждый процессор BlueGene [даётся](http://hpc.cmc.msu.ru/bgp) 2 Gb оперативной памяти, т.е. 2^31^ байта помножить на число запрошенных вычислителей. У RMAT графа 2^N^ степени обычно 2^N+5^ рёбер (если пользоваться generator'ом). Нехитрая математика показывает, что 2^18^ вершин и 2^23^ рёбер не являются неподъёмным числом для суперкомпа, даже такого старого как BlueGene.

# Acknowledgements

- Шейдаев Вадим — автор мануала.
- Полушкин Алексей — помогал мне прорываться сквозь дебри статического плюсового полиморфизма.
- Тутельян Сергей — ценные замечания по поводу ошибок компиляции.
- Афанасьев Никита — составил мне компанию в безуспешных попытках собрать Boost.
- Королёв Николай — поделился ссылкой на кошерно собранный Boost и нужными флагами линковки.
- Аят Оспанов — нашёл ошибку в команде компиляции программы под BlueGene и упростил компиляцию `generator.cpp`.
- 9-ый студент 615 группы 2016 года. Скомпилил для нас буст и сэкономил нам кучу времени.

