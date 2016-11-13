# Глава 64. Boost.Serialization

## Содержание
- [Архив](#Архив)
- [Указатели и ссылки](#Указатели-и-ссылки)
- [Сериализация объектов иерархии классов](#Сериализация-объектов-иерархии-классов)
- [Оберточные функции для оптимизации](#Оберточные-функции-для-оптимизации)

Библиотека [Boost.Serialization](www.boost.org/doc/libs/1_62_0/libs/serialization/doc/index.html) позволяет преобразовать объекты в программе C++ в последовательности байтов, которые могут быть сохранены и загружены для восстановления объектов. Существуют различные форматы данных, доступные для определения правил генерации последовательности байтов. Все форматы, поддерживаемые Boost.Serialization предназначены только для использования с этой библиотекой. Например, формат XML, разработанный для Boost.Serialization, не должен использоваться для обмена данными с программами, которые не используют Boost.Serialization. Единственное преимущество формата XML является то, что он может сделать отладку проще, так как объекты C++ сохраняются в удобном для чтения формате.

Как указано в [release notes](www.boost.org/users/history/version_1_55_0.html) версии 1.55.0 библиотек Boost, отсутствие include вызывает ошибку компилятора в Visual C++ 2013. Эта ошибка была исправлена в Boost 1.56.0.

***

# Архив

Основной концепцией Boost.Serialization является архив. Архив - это последовательность байтов, которая представляет собой сериализованные C++ объекты. Они могут быть добавлены в архив для сериализации, а затем загружены из архива. При восстановлении ранее сохраненных объектов C++, предполагаются одинаковые типы.

<a name="example641"></a>
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
Boost.Serialization включает в себя архивные классы, например **`Boost::archive::text_oarchive`**, который определен в файле **`Boost/archive/text_oarchive.hpp`**. Этот класс дает возможность сериализовать объекты в текстовый поток. Используя Boost 1.56.0, [пример 64.1](#example641) выведет **`22 serialization::archive 11 1`** в стандартный поток вывода.

Как видно, объект **`oa`** типа **`Boost::archive::text_oarchive`** может быть использован как поток для сериализации переменной при помощи оператора **`<<`**. Однако архивы не должны рассматриваться как обычные потоки, храненящие произвольные данные. Для восстановления данных Вы должны открыть его так, как и сохранили, используя те же типы данных в том же порядке. [Пример 64.2](#example642) сериализует и возвращает переменную типа int.

<a name="example642"></a>
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

Класс **`boost::archive::text_oarchive`** сериализует данные как текстовый поток, и класс **`boost::archive::text_iarchive`** извлекает данные как из текстового потока. Чтобы использовать эти классы, неодходимо подключить заголовочные файлы **`boost/archive/text_iarchive.hpp`** и **`boost/archive/text_oarchive.hpp`**.

Конструкторы архивов принимают входной или выходной поток в качестве параметра. Поток используется для сериализации и восстановления данных. Хотя [пример 64.2](#example642) обращается к файлу, другие потоки, такие как stringstream, также могут быть использованы.

<a name="example643"></a>
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
[Пример 64.3](#example643) записывает **`1`** в стандартный поток вывода, используя stringstream для сериализации данных.

Пока что были сериализованы только простые типы данных. [Пример 64.4](#example644) показывает, как сериализовать объекты типов, определяемых пользователем.

<a name="example644"></a>
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
Для того, чтобы сериализовать объекты типов, определяемых пользователем, необходимо определить функцию-член **`serialize()`**. Эта функция вызывается, когда объект сериализуется или же восстанавливается из потока байтов. Так как **`serialize()`** используется как для сериализации, так и для восстановления, Boost.Serialization поддерживает оператор **`&`** в дополнение к операторам **`<<`** и **`>>`**. С оператором **`&`** нет необходимости различать сериализацию и восстановление в **`serialize()`**.

Функция **`serialize()`** автоматически вызывается каждый раз, когда объект сериализован или восстановлен. Она никогда не должна вызываться явным образом, и поэтому должна быть приватной. Если объявлена как private, класс **`boost::serialization::access`** должен быть объявлен дружественным, чтобы дать Boost.Serialization доступ к функциям-членам.

Возможны ситуации, которые не позволяют изменить существующий класс, чтобы добавить фунцкию serialize(). Например, это касается классов из стандартной библиотеки шаблонов.

<a name="example645"></a>
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
Для того, чтобы сериализовать типы, которые не могут быть изменены, можно определить свободно стоящую функцию **`serialize()`**, как показано в [примере 64.5](#example645). Эта функция принимает ссылку на объект соответствующего типа в качестве второго параметра.

Реализация serialize() как свободно стоящей функции требует, чтобы основные переменные класса были доступны извне. В [примере 64.5](#example645) **`serialize()`** может определяться только как свободно стоящая функция, поскольку **`legs_`** не является приватной переменной класса **`animal`**.

Boost.Serialization предоставляет функцию **`serialize()`** для многих классов из стандартной библиотеки. Для сериализации объектов, основанных на стандартных классах, должны быть включены дополнительные заголовочные файлы.

<a name="example646"></a>
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
[Пример 64.6](#example646) расширяет класс **`animal`** путем добавления члена **`name_`** - переменной типа **`std::string`**. Для того, чтобы сериализовать эту переменную, должен быть подключен заголовочный файл **`boost/serialization/string.hpp`**, чтобы предоставить свободно стоящую функцию **`serialize()`**.

Как упоминалось ранее, Boost.Serialization определяет функции **`serialize()`** для многих классов из стандартной библиотеки. Эти функции определены в заголовочных файлах, имеющих то же имя, что и соответствующие файлы заголовков из стандарта. Таким образом, чтобы сериализовать объекты типа **`std::string`**, подключают файл **`boost/serialization/string.hpp`**, а чтобы сериализовать объекты типа **`std::vector`**, подключают файл **`boost/serialization/vector.hpp`**. Подключить необходимый заголовочный файл не составит труда.

Параметр функции **`serialize()`**, который был проигнорирован, является **`version`**. Этот параметр помогает делать архивы с обратной совместимостью. [Пример 64.7](#example647) может загрузить архив, который был создан в [примере 64.5](#example645). Версия класса **`animal`** в [примере 64.5](#example645) не содержит **`name_`**. [Пример 64.7](#example647) проверяет номер версии при загрузке архива и только обращается к имени, если версия больше 0. Это позволяет обрабатывать старые архивы, которые были созданы без имени.

<a name="example647"></a>
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
Макрос **`BOOST_CLASS_VERSION`** присваивает классу номер версии. Для класса **`animal`** в [примере 64.7](#example647) — 1. Если **`BOOST_CLASS_VERSION`** не используется, по умолчанию номер версии равен 0.

Номер версии хранится в архиве и является его частью. В то время как номер версии, указанный для определенного класса через макрос **`BOOST_CLASS_VERSION`**, используется во время сериализации, параметру **`version`** функции **`serialize()`** будет присвоено значение, хранящееся в архиве, при восстановлении. Если новая версия класса **`animal`** обращается к архив, содержащий объект, сериализованный со старой версией, переменная **`name_`** не будет быть восстановлена, так как старая версия не имеет такого члена.

***

# Указатели и ссылки

[Boost.Serialization](www.boost.org/doc/libs/1_62_0/libs/serialization/doc/index.html) также обеспечивает работу с указателями и ссылками. Так как указатель хранит адрес объекта, сериализовывать адрес нет смысла. При сериализации указателей и ссылок сериализуется объект, на который ссылается указатель или ссылка.

<a name="example648"></a>
`Пример 64.8. Сериализация указателей`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <iostream>
#include <sstream>

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
  boost::archive::text_oarchive oa{ss};
  animal *a = new animal{4};
  oa << a;
  std::cout << std::hex << a << '\n';
  delete a;
}

void load()
{
  boost::archive::text_iarchive ia{ss};
  animal *a;
  ia >> a;
  std::cout << std::hex << a << '\n';
  std::cout << std::dec << a->legs() << '\n';
  delete a;
}

int main()
{
  save();
  load();
}
```

[Пример 64.8](#example648) создает объект типа **`animal`** с помощью **`new`** и присваивает его указателю **`a`**. Указатель - не **`*a`** - становится сериализованным. Boost.Serialization автоматически сериализует объект, на который ссылается указатель **`a`**, а не адрес этого объекта.

После восстановления архива, **`a`** не обязаnельно будет хранить тот же адрес. Будет создан новый объект, и его адрес будет присвоен **`a`** вместо прежнего. Boost.Serialization гарантирует только то, что объект остался таким же, каким был до сериализации, но не гарантирует сохранение адреса.

Так как умные указатели работают с динамически выделяемой памятью, Boost.Serialization также обеспечивает поддержку и для них.

<a name="example649"></a>
`Пример 64.9. Сериализация умных указателей`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <boost/serialization/scoped_ptr.hpp>
#include <boost/scoped_ptr.hpp>
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
  boost::scoped_ptr<animal> a{new animal{4}};
  oa << a;
}

void load()
{
  text_iarchive ia{ss};
  boost::scoped_ptr<animal> a;
  ia >> a;
  std::cout << a->legs() << '\n';
}

int main()
{
  save();
  load();
}
```
[Пример 64.9](#example649) использует умный указатель **`boost::scoped_ptr`** для управления динамически выдленным объектом типа **`animal`**. Для сериализации такого указателя подключите **`boost/serialization/scoped_ptr.hpp`**. А для сериализации умного указателя типа **`boost::shared_ptr`** подключите заголовочный файл **`boost/serialization/shared_ptr.hpp`**.

Пожалуйста, обратите внимание, что Boost.Serialization пока не обновлен до C++11. Умные указатели из стандартной библиотеки C++11, такие как **`std::shared_ptr`** и **`std::unique_ptr`**, в настоящее время не поддерживаются.

<a name="example6410"></a>
`Пример 64.10. Сериализация ссылки`
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
  animal &r = a;
  oa << r;
}

void load()
{
  text_iarchive ia{ss};
  animal a;
  animal &r = a;
  ia >> r;
  std::cout << r.legs() << '\n';
}

int main()
{
  save();
  load();
}
```
Используя Boost.Serialization, можно сериализовать ссылки без проблем (см. [пример 64.10](#example6410)). Как и с указателями, указанный объект сериализуется автоматически.

***

# Сериализация объектов иерархии классов

Производные классы должны иметь доступ к функции **`boost::serialization::base_object()`** внутри функции-члена **`serialize()`** для сериализации объектов, основанных на иерархии классов. Данная функция гарантирует правильную сериализацию наследуемых переменных из базового класса.

<a name="example6411"></a>
`Пример 64.11. Правильная сериализация производных классов`
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

class bird : public animal
{
public:
  bird() = default;
  bird(int legs, bool can_fly) :
    animal{legs}, can_fly_{can_fly} {}
  bool can_fly() const { return can_fly_; }

private:
  friend class boost::serialization::access;

  template <typename Archive>
  void serialize(Archive &ar, const unsigned int version)
  {
    ar & boost::serialization::base_object<animal>(*this);
    ar & can_fly_;
  }

  bool can_fly_;
};

void save()
{
  text_oarchive oa{ss};
  bird penguin{2, false};
  oa << penguin;
}

void load()
{
  text_iarchive ia{ss};
  bird penguin;
  ia >> penguin;
  std::cout << penguin.legs() << '\n';
  std::cout << std::boolalpha << penguin.can_fly() << '\n';
}

int main()
{
  save();
  load();
}
```
[Пример 64.11](#example6411) использеут класс **`bird`**, наследованный от **`animal`**. В обоих классах определена приватная функция **`serialize()`**, что позволяет серилиазовать объекты, основанные на любом классе. Так как класс **`bird`** наследован от класса **`animal`**, **`serialize()`** должна убедиться, что переменные, наследованные от **`animal`** также были сериализованы.

Унаследованные переменные-члены сериализуются благодаря обращению к базовому классу через функцию **`serialize()`** производного класса и вызова **`boost::serialization::base_object()`**. Необходимо вызывать эту функцию, а не, например, **`static_cast`**, потому что **`boost::serialization::base_object()`** обеспечиват правильную сериализацию.

Адреса динамически созданных объектов могут быть присвоены указателям соответсвующего типа базового класса. [Пример 64.12](#example6412) показывает, что Boost.Serialization правильно их сериализует.

<a name="example6412"></a>
`Пример 64.12. Статическая регистрация производных классов с **BOOST_CLASS_EXPORT**`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <boost/serialization/export.hpp>
#include <iostream>
#include <sstream>

using namespace boost::archive;

std::stringstream ss;

class animal
{
public:
  animal() = default;
  animal(int legs) : legs_{legs} {}
  virtual int legs() const { return legs_; }
  virtual ~animal() = default;

private:
  friend class boost::serialization::access;

  template <typename Archive>
  void serialize(Archive &ar, const unsigned int version) { ar & legs_; }

  int legs_;
};

class bird : public animal
{
public:
  bird() = default;
  bird(int legs, bool can_fly) :
    animal{legs}, can_fly_{can_fly} {}
  bool can_fly() const { return can_fly_; }

private:
  friend class boost::serialization::access;

  template <typename Archive>
  void serialize(Archive &ar, const unsigned int version)
  {
    ar & boost::serialization::base_object<animal>(*this);
    ar & can_fly_;
  }

  bool can_fly_;
};

BOOST_CLASS_EXPORT(bird)

void save()
{
  text_oarchive oa{ss};
  animal *a = new bird{2, false};
  oa << a;
  delete a;
}

void load()
{
  text_iarchive ia{ss};
  animal *a;
  ia >> a;
  std::cout << a->legs() << '\n';
  delete a;
}

int main()
{
  save();
  load();
}
```
Эта программа создает объект типа **`bird`** внутри функции **`save()`** и присваивает его указателю типа **`animal*`**, который, в свою очередь, сериализуется через оператор **`>>`**.

Как было упомянуто в предыдущем разделе, сериализуется объект, на который ссылаются, а не указатель. Чтобы Boost.Serialization распознал, что необходимо сериализовать объект типа **`bird`**, несмотря на то, что указатель имеет тип **`animal*`**, класс **`bird`** должен быть объявлен. Это делается с помощью макроса **`BOOST_CLASS_EXPORT`**, который определен в файле **`boost/serialization/export.hpp`**. Так как тип **`bird`** не появляется в определении указателя, Boost.Serialization не может серилиазовать тип **`bird`** правильно.

Макрос **`BOOST_CLASS_EXPORT`** необходимо использовать, когда объект производного класса сериализуется при помощи указателя на базовый класс. Недостатком **`BOOST_CLASS_EXPORT`** является то, что из-за статической регистрации, зарегистрированные классы могут использовать ся для сериализации не полностью. Boost.Serialization предлагает решения для этого сценария.

<a name="example6413"></a>
`Пример 64.13. Регистрация динамических производных классов с register_type()`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <boost/serialization/export.hpp>
#include <iostream>
#include <sstream>

std::stringstream ss;

class animal
{
public:
  animal() = default;
  animal(int legs) : legs_{legs} {}
  virtual int legs() const { return legs_; }
  virtual ~animal() = default;

private:
  friend class boost::serialization::access;

  template <typename Archive>
  void serialize(Archive &ar, const unsigned int version) { ar & legs_; }

  int legs_;
};

class bird : public animal
{
public:
  bird() = default;
  bird(int legs, bool can_fly) :
    animal{legs}, can_fly_{can_fly} {}
  bool can_fly() const { return can_fly_; }

private:
  friend class boost::serialization::access;

  template <typename Archive>
  void serialize(Archive &ar, const unsigned int version)
  {
    ar & boost::serialization::base_object<animal>(*this);
    ar & can_fly_;
  }

  bool can_fly_;
};

void save()
{
  boost::archive::text_oarchive oa{ss};
  oa.register_type<bird>();
  animal *a = new bird{2, false};
  oa << a;
  delete a;
}

void load()
{
  boost::archive::text_iarchive ia{ss};
  ia.register_type<bird>();
  animal *a;
  ia >> a;
  std::cout << a->legs() << '\n';
  delete a;
}

int main()
{
  save();
  load();
}
```
Вместо того, чтобы использовать макрос **`BOOST_CLASS_EXPORT`**, [пример 64.13](#example6413) вызывает шаблонную функцию-член **`register_type()`**. Тип регистрации передается в качетсве параметра. Обратите внимание, что **`register_type()`** должна быть вызвана как в **`save()`**, так и в **`load()`**.

Преимуществом **`register_type()`** является то, что необходмо регистрировать только те классы, которые сериализуются. Например, при разработке библиотеки, никто не знает, какие классы раработчик будет использовать для сериализации позже. В то время как макрос **`BOOST_CLASS_EXPORT`** упрощает этот процесс, он сожет регистрировать типы, которые не будут использоваться при сериализации.

***

## Оберточные функции для оптимизации

В этом разделе представлены оберточные функции, которые оптимизируют процесс сериализации. Эти функции помечают объекты, чтобы позволить Boost.Serialization применять некоторые методы оптимизации.

<a name="example6414"></a>
`Пример 64.14. Сериализация массива без оберточной функции`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <boost/array.hpp>
#include <iostream>
#include <sstream>

using namespace boost::archive;

std::stringstream ss;

void save()
{
  text_oarchive oa{ss};
  boost::array<int, 3> a{{0, 1, 2}};
  oa << a;
}

void load()
{
  text_iarchive ia{ss};
  boost::array<int, 3> a;
  ia >> a;
  std::cout << a[0] << ", " << a[1] << ", " << a[2] << '\n';
}

int main()
{
  save();
  load();
}
```
[Пример 64.14](#example6414) использует Boost.Serialization без оберточной функции. Пример создает и записывает значения: **`22 serialization::archive 11 0 0 3 0 1 2`** в строку. Используя оберточную функцию **`boost::serialization::make_array()`**, значение может сокращено: **`22 serialization::archive 11 0 1 2`**.

<a name="example6415"></a>
`Пример 64.15. Сериализация массива с оберточной функцией make_array()`
```c++
#include <boost/archive/text_oarchive.hpp>
#include <boost/archive/text_iarchive.hpp>
#include <boost/serialization/array.hpp>
#include <array>
#include <iostream>
#include <sstream>

using namespace boost::archive;

std::stringstream ss;

void save()
{
  text_oarchive oa{ss};
  std::array<int, 3> a{{0, 1, 2}};
  oa << boost::serialization::make_array(a.data(), a.size());
}

void load()
{
  text_iarchive ia{ss};
  std::array<int, 3> a;
  ia >> boost::serialization::make_array(a.data(), a.size());
  std::cout << a[0] << ", " << a[1] << ", " << a[2] << '\n';
}

int main()
{
  save();
  load();
}
```
**`boost::serialization::make_array()`** принимает адрес и длину массива. Однако, так как заранее известно, длина не должна быть сериализована как часть массива.

**`boost::serialization::make_array()`** может использоваться всякий раз, когда контейнеры, такие как **` std::array`** или **`std::vector`** , содержат массив, который необходимо непосредственно сериализовать. Обычные переманные, корторые обычно тоже сериализуются, пропускаются (см. [пример 64.15](#example6415)).

Boost.Serialization также предоставляет оберточные функции **`boost::serialization::make_binary_object()`**. Аналогично **`boost::serialization::make_array()`**, она принимает адрес и длину массива. **`boost::serialization::make_binary_object()`** используется исключительно для бинарных данных, которые не имеют базовой структры, в то время как **`boost::serialization::make_array()`** используется для массивов.
