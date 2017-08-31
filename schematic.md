---
title: Schematic - Level Up your DSL
author: Денис Редозубов, @rufuse, typeable.io(@typeableIO)
patat:
    incrementalLists: true
    wrap: true
    theme:
        emph: [vividBlue, onVividBlack, bold]
        imageTarget: [onDullWhite, vividRed]
...

# Ccылки

https://github.com/dredozubov/fprog17

https://github.com/typeable/schematic

---

# Спецификация

- Тесты
- Типы
- Внешние документы

---

# Typelevel-программирование

Повышение точности спецификации на уровне типов. За счет того что система типов является частью компилятора, это отсекает неверные конструкции в тексте программы.

---

# DSL

Domain-specific language. Для нашего примера доменной областью будет формат JSON.
С помощью типов опишем область допустимых значений для JSON структур.

---

# DSL

```
type Book = SchemaObject
  '[ '("book_name", SchemaText '[])
   , '("rating", 'SchemaOptional (SchemaNumber '[NLe 10, NGt 0]))]
```

---

# Проблематика

API или JSON-storage представляют из себя транспортные слои, от целостности которых зависит работоспособность системы, поэтому менять их сложно из-за возможной несовсместимости с более ранними версиями.

---

# Как это решают

- Прилежание и отвага
- Тесты
- Отделение транспортного слоя для передачи данных от бизнес-логики
- Явное версионирование
- Миграции

---

# Как это выглядит в haskell-проекте

- Куча типов, описывающих транспортный слой. Как правило - рекорды, содержащие текстовые и численные поля
- Конвертеры из типов для бизнес-логики в транспортные и обратно
- Код скучный, допустить ошибку легко

---

# Type-level schema

- Нет нужды описывать транспортные типы руками - они полностью определены схемой
- `encode (decode x) = x` и `decode (encode s) = s`
- Схема является подмножеством json-schema, поэтому возможен экспорт (draft4)
- Есть возможность описать дифф между несколькими схемами, что дает возможность реализовать миграции
- Ошибки проявляют себя во время компиляции, а не в рантайме!
- Один источник правды для всех аспектов кода, связанных с JSON API

---

# Type-level schema


```
data Schema
  = SchemaText [TextConstraint]
  | SchemaBoolean
  | SchemaNumber [NumberConstraint]
  | SchemaObject [(Symbol, Schema)]
  | SchemaArray [ArrayConstraint] Schema
  | SchemaNull
  | SchemaOptional Schema
```

---

# Не все можно проверить во время компиляции

Schematic генерирует runtime-валидаторы для данных транспортного слоя:

```
data TextConstraint
  = TEq Nat
  | TLt Nat
  | TLe Nat
  | TGt Nat
  | TGe Nat
  | TRegex Symbol
  | TEnum [Symbol]
  deriving (Generic)
```

---

# Не все можно проверить во время компиляции

Для валидации используется библиотека validationt

В самой схеме это выглядит так:

```
SchemaText '[ 'TEnum '["foo", "bar"] ])
```

---

# Транспортные типы

```
data JsonRepr :: Schema -> Type where
  ReprText :: Text -> JsonRepr ('SchemaText cs)
  ReprNumber :: Scientific -> JsonRepr ('SchemaNumber cs)
  ReprBoolean :: Bool -> JsonRepr 'SchemaBoolean
  ReprNull :: JsonRepr 'SchemaNull
  ReprArray :: V.Vector (JsonRepr s) -> JsonRepr ('SchemaArray cs s)
  ReprObject :: Rec FieldRepr fs -> JsonRepr ('SchemaObject fs)
  ReprOptional :: Maybe (JsonRepr s) -> JsonRepr ('SchemaOptional s)
```

---

# Миграции

Миграция это схема структурных преобразований json-документа, каждое из которых можно представить парой из json-path, т.е. пути до элемента в структуре и самого преобразования:

```
Diff '[ 'PKey "bar" ] ('Update ('SchemaText '[])
Diff '[ 'PKey "foo" ] ('Update ('SchemaNumber '[]
```

---

# Миграции

Миграция целиком это всего лишь серия последовательных преобразований схемы:

```
Migration "migration_name"
 '[ 'Diff '[ PKey "bar" ] (Update (SchemaText '[]))
  , 'Diff '[ PKey "foo" ] (Update (SchemaNumber '[]))
  ]

```

---

# Миграции

Последовательность миграций дает возможность описать все версии транспортного типа:

```
Versioned BookSchema '[ AddAuthor, AddReviews ]
```

---

# Линзы

Поддержка геттеров и сеттеров для гетерогенных структур включена в библиотеку

```
fget (Proxy @"foo") objectData
fput newFooVal objectData
```

Поддерживается библиотека lens:

```
objectData ^. flens (Proxy @"foo")
set (flens (Proxy @"foo")) newFooVal objectData
```

---

# Как это все работает?

- TypeLiterals
- Typeclasses
- Type families
- Singleton types
- GADTs
- TypeInType
- щепотка магии

---

# Как это можно использовать?

* JSONRPC APIs
* Хранилища с поддержкой json - миграции и валидация данных может осуществляться на уровне приложения, это позволяет строить event sourcing системы и многое другое.
* Версионированные конфиги

---

# Как будет развиваться schematic

- Генераторы данных для тестирования и генерации примеров well-formed запросов и, возможно, схем
- Поддержка рекурсивных схем
- Версионированные HTTP API и возможная интеграция с servant
- поддержка swagger

---

# Выводы

- Описания на уровне типов позволяют получить generic код бесплатно
- исключить обработку неверных кейсов кодом библиотеки
- Взгляд на проблему с новой стороны
