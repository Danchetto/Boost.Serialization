# Глава 64. Boost.Serialization

## Содержание
- Архив
- Указатели и ссылки
- Сериализация иерархии классов объектов
- Оберточные функции для оптимизации

Библиотека Boost.Serialization позволяет преобразовать объекты в программе C++ в последовательности байтов, которые могут быть сохранены и загружены для восстановления объектов. Существуют различные форматы данных, доступные для определения правил генерации последовательности байтов. Все форматы, поддерживаемые Boost.Serialization предназначены только для использования с этой библиотекой. Например, формат XML, разработанный для Boost.Serialization, не должен использоваться для обмена данными с программами, которые не используют Boost.Serialization. Единственное преимущество формата XML является то, что он может сделать отладку проще, так как объекты C++ сохраняются в удобном для чтения формате.

***

# Архив

Основной концепцией Boost.Serialization является архив. Архив - это последовательность байтов, которая представляет собой сериализованные C++ объекты. Они могут быть добавлены в архив для сериализации, а затем загружены из архива. Для того, чтобы восстановить ранее сохраненные объекты C++, предполагаются одинаковые типы.

`Пример 64.1. Используется boost::archive::text_oarchive`
``` c++
#include <boost/archive/text_oarchive.hpp>
#include <iostream>

using namespace boost::archive;

int main()
{
  text_oarchive oa{std::cout};
  int i = 1;
  oa << i;
}
```
Boost.Serialization включает в себя архивные классы, например Boost::archive::text_oarchive, который определен в файле Boost/archive/text_oarchive.hpp. Этот класс дает возможность сериализовать объекты в текстовый поток. Используя Boost 1.56.0, пример 64.1 выведет **22 serialization::archive 11 1** в стандартный выходной поток.

Как видно, объект oa типа Boost::archive::text_oarchive может быть использован как поток для сериализации переменной при помощи оператора <<. Однако архивы не должны рассматриваться как обычные потоки, храненящие произвольные данные. Для восстановления данных, Вы должны открыть его, как вы храните его, используя те же типы данных в том же порядке. Пример 64.2 сериализует и возвращает переменную типа int.

`Пример 64.2. Используется boost::archive::text_iarchive`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <iostream>
#include <fstream>

using namespace boost::archive;

void save()
{
  std::ofstream file{"archive.txt"};
  text_oarchive oa{file};
  int i = 1;
  oa << i;
}

void load()
{
  std::ifstream file{"archive.txt"};
  text_iarchive ia{file};
  int i = 0;
  ia >> i;
  std::cout << i << '\n';
}

int main()
{
  save();
  load();
}
```

Класс boost::archive::text_oarchive сериализует данные как текстовый поток, и класс boost::archive::text_iarchive извлекает данные как из текстового потока. Чтобы использовать эти классы, неодходимо подключить заголовочные файлы boost/archive/text_iarchive.hpp и boost/archive/text_oarchive.hpp.

Конструкторы архивов принимают входной или выходной поток как параметр. Поток используется для сериализации и восстановления данных. Хотя пример 64.2 обращается к файлу, другие потоки, такие как stringstream, также могут использоваться.

`Пример 64.3. Сериализация с stringstream`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <iostream>
#include <sstream>

using namespace boost::archive;

std::stringstream ss;

void save()
{
  text_oarchive oa{ss};
  int i = 1;
  oa << i;
}

void load()
{
  text_iarchive ia{ss};
  int i = 0;
  ia >> i;
  std::cout << i << '\n';
}

int main()
{
  save();
  load();
}
```
Пример 64.3 записывает **1** в стандартный поток вывода, используя stringstream для сериализации данных.

Пока что были сериализованы только простые типы данных. Пример 64.4 показывает, как сериализовать объекты типов, определяемых пользователем.

`Пример 64.4. Сериализация определяемых пользователем типов с функцией-членом`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <iostream>
#include <sstream>

using namespace boost::archive;

std::stringstream ss;

class animal
{
public:
  animal() = default;
  animal(int legs) : legs_{legs} {}
  int legs() const { return legs_; }

private:
  friend class boost::serialization::access;

  template <typename Archive>
  void serialize(Archive &ar, const unsigned int version) { ar & legs_; }

  int legs_;
};

void save()
{
  text_oarchive oa{ss};
  animal a{4};
  oa << a;
}

void load()
{
  text_iarchive ia{ss};
  animal a;
  ia >> a;
  std::cout << a.legs() << '\n';
}

int main()
{
  save();
  load();
}
```
Для того, чтобы сериализовать объекты типов, определяемых пользователем, необходимо определить функцию-член serialize(). Эта функция вызывается, когда объект сериализуется или же восстанавливается из потока байтов. Так как serialize() используется как для сериализации, так и для восстановления, Boost.Serialization поддерживает оператор & в дополнение к операторам << и >>. С оператором & нет необходимости различать сериализацию и восстановление в serialize().

Функция serialize() автоматически вызывается каждый раз, когда объект сериализован или восстановлен. Она никогда не должна вызываться явным образом, и поэтому должна быть приватной. Если объявлена как private, класс boost::serialization::access должен быть объявлен дружественным, чтобы дать Boost.Serialization доступ к функциям-членам.

Возможны ситуации, которые не позволяют изменить существующий класс, чтобы добавить фунцкию serialize(). Например, это касается классов из стандартной библиотеки шаблонов.

`Пример 64.5. Сериализация с свободно стоящей функцией`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <iostream>
#include <sstream>

using namespace boost::archive;

std::stringstream ss;

struct animal
{
  int legs_;

  animal() = default;
  animal(int legs) : legs_{legs} {}
  int legs() const { return legs_; }
};

template <typename Archive>
void serialize(Archive &ar, animal &a, const unsigned int version)
{
  ar & a.legs_;
}

void save()
{
  text_oarchive oa{ss};
  animal a{4};
  oa << a;
}

void load()
{
  text_iarchive ia{ss};
  animal a;
  ia >> a;
  std::cout << a.legs() << '\n';
}

int main()
{
  save();
  load();
}
```
Для того, чтобы сериализовать типы, которые не могут быть изменены, можно определить свободно стоящую функцию serialize(), как показано в примере 64.5. Эта функция принимает ссылку на объект соответствующего типа в качестве второго параметра.

Реализация serialize() как свободно стоящей функции требует, чтобы основные переменные класса были доступны извне. В примере 64.5 serialize() может определяться только как свободно стоящая функция, поскольку **`legs_`** не является приватной переменной класса **`animal`**.

Boost.Serialization предоставляет функцию serialize() для многих классов из стандартной библиотеки. Для сериализации объектов, основанных на стандартных классах, должны быть включены дополнительные заголовочные файлы.

`Пример 64.6. Сериализация строк`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <boost/serialization/string.hpp>
#include <iostream>
#include <sstream>
#include <string>
#include <utility>

using namespace boost::archive;

std::stringstream ss;

class animal
{
public:
  animal() = default;
  animal(int legs, std::string name) :
    legs_{legs}, name_{std::move(name)} {}
  int legs() const { return legs_; }
  const std::string &name() const { return name_; }

private:
  friend class boost::serialization::access;

  template <typename Archive>
  friend void serialize(Archive &ar, animal &a, const unsigned int version);

  int legs_;
  std::string name_;
};

template <typename Archive>
void serialize(Archive &ar, animal &a, const unsigned int version)
{
  ar & a.legs_;
  ar & a.name_;
}

void save()
{
  text_oarchive oa{ss};
  animal a{4, "cat"};
  oa << a;
}

void load()
{
  text_iarchive ia{ss};
  animal a;
  ia >> a;
  std::cout << a.legs() << '\n';
  std::cout << a.name() << '\n';
}

int main()
{
  save();
  load();
}
```
Пример 64.6 расширяет класс **`animal`** путем добавления члена **`name_`**, переменной типа std::string. Для того, чтобы сериализовать эту переменную, должен быть подключен заголовочный файл boost/serialization/string.hpp, чтобы предоставить свободно стоящую функцию serialize().

Как упоминалось ранее, Boost.Serialization определяет функции serialize() для многих классов из стандартной библиотеки. Эти функции определены в заголовочных файлах, имеющих то же имя, что и соответствующие файлы заголовков из стандарта. Таким образом, чтобы сериализовать объекты типа std::string, подключают файл boost/serialization/string.hpp, а чтобы сериализовать объекты типа std::vector, подключают файл boost/serialization/vector.hpp . Это довольно очевидно, какой заголовочный файл необходимо подключить.

Один параметр функции serialize(), который был проигнорирован, является **`version`**. Этот параметр помогает делать архивы с обратной совместимостью. Пример 64.7 может загрузить архив, который был создан в примере 64.5. Версия класса**`animal`** в примере 64.5 не содержит **`name_`**. Пример 64.7 проверяет номер версии при загрузке архива и только обращается к имени, если версия больше 0. Это позволяет обрабатывать старые архивы, который были созданы без имени.

`Пример 64.7. Обратная совместимость с номерами версий`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <boost/serialization/string.hpp>
#include <iostream>
#include <sstream>
#include <string>
#include <utility>

using namespace boost::archive;

std::stringstream ss;

class animal
{
public:
  animal() = default;
  animal(int legs, std::string name) :
    legs_{legs}, name_{std::move(name)} {}
  int legs() const { return legs_; }
  const std::string &name() const { return name_; }

private:
  friend class boost::serialization::access;

  template <typename Archive>
  friend void serialize(Archive &ar, animal &a, const unsigned int version);

  int legs_;
  std::string name_;
};

template <typename Archive>
void serialize(Archive &ar, animal &a, const unsigned int version)
{
  ar & a.legs_;
  if (version > 0)
    ar & a.name_;
}

BOOST_CLASS_VERSION(animal, 1)

void save()
{
  text_oarchive oa{ss};
  animal a{4, "cat"};
  oa << a;
}

void load()
{
  text_iarchive ia{ss};
  animal a;
  ia >> a;
  std::cout << a.legs() << '\n';
  std::cout << a.name() << '\n';
}

int main()
{
  save();
  load();
}
```
Макрос BOOST_CLASS_VERSION присваивает классу номер версии. Для класса **`animal`** в примере 64.7 — 1. Если BOOST_CLASS_VERSION не используется, по умолчанию номер версии равен 0.

Номер версии хранится в архиве и является его частью. В то время как номер версии, указанный для определенного класса через макрос BOOST_CLASS_VERSION, используется во время сериализации, параметру **`version`** функции serialize() будет присвоено значение, хранящееся в архиве, при восстановлении. Если новая версия класса **`animal`** обращается к архив, содержащий объект, сериализованный со старой версией, переменная **`name_`** не будет быть восстановлена, так как старая версия не имеет такого члена.
