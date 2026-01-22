![](img/litros-logo-horizontal.png)

<h1 align="center">Litros<br>Library Tracker for OneScript</br></h1>

<p align="center">
<a href="https://github.com/alei1180/litros/blob/main/LICENSE"><img alt="License" src="https://img.shields.io/github/license/alei1180/litros?style=badge"></a>
<a href="https://github.com/alei1180/litros/issues"><img alt="GitHub issues" src="https://img.shields.io/github/issues-raw/alei1180/litros?style=badge"></a>
<a href="https://github.com/alei1180/litros/releases/latest"><img alt="Last release" src="https://img.shields.io/github/v/release/alei1180/litros?include_prereleases&label=last%20release&style=badge"></a>
<a href="https://sonar.openbsl.ru/dashboard?id=litros"><img alt="SonarQube: quality gate" src="https://sonar.openbsl.ru/api/project_badges/measure?project=litros&metric=alert_status&token=sqb_174d3352e142da6217583afe4bfbd7af29ee137d"></a>
<a href="https://sonar.openbsl.ru/dashboard?id=litros"><img alt="SonarQube: coverage" src="https://sonar.openbsl.ru/api/project_badges/measure?project=litros&metric=coverage&token=sqb_174d3352e142da6217583afe4bfbd7af29ee137d"></a>
</p>

## Назначение

`Litros` - трэкер библиотек `OneScript` на `GitHub`:

- Показывает информацию об использовании библиотеки (пакета) в других проектах `OneScript` на `GitHub`
- Генерирует бэйджик о количестве использований для вставки в `markdown` файлы

## Сайт

[litrosbadges.ru](https://litrosbadges.ru/)

## Установка

```shell
opm install litros
```

## Использование

## Библиотека

### OneScript

```bsl
#Использовать litros

Сканер = Новый СканерИспользованийПакета("ТокенГитхаб");
РезультатПоиска = Сканер.НайтиРепозитории("ИмяПакета"); // Имя пакета, которое определено через инструкцию #Использовать
```

### OneScript + [Autumn](https://github.com/autumn-library/autumn)

[Сгенерировать токен](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token) в настройках профиля `GitHub` и указать его в файле `autumn-properties.json` в корне вашего проекта:
```bsl
{
    "litros": {
        "ТокенGitHub": "ВАШ-ТОКЕН"
    }
}
```

```bsl
#Использовать litros

&Пластилин 
Перем СканерИспользованийПакета;

&Желудь
Процедура ПриСозданииОбъекта()

КонецПроцедуры

Процедура ПолезнаяНагрузка() Экспорт
 РезультатПоиска = СканерИспользованийПакета.НайтиРепозитории("PACKAGE_NAME");
КонецПроцедуры
```

## Публичный интерфейс

### Класс СканерИспользованийПакета

#### НайтиРепозитории

```bsl
// Выполняет поиск репозиториев GitHub, в исходном коде которых
// обнаружено подключение указанного пакета OneScript.
//
// Параметры:
//   ИмяПакета - Строка - Имя анализируемого пакета
//
// Возвращаемое значение:
//   Структура - Результат выполнения поиска со следующими полями:
//     * Данные - Массив из Структура - Список уникальных репозиториев, в которых найдено использование пакета:
//       ** URL       - Строка - URL репозитория
//       ** ПолноеИмя - Строка - Полное имя репозитория (owner/name)
//       ** Владелец  - Строка - Логин владельца
//       ** ЭтоФорк   - Булево - Признак форка
//     * Успех     - Булево - Признак отсутствия ошибок при выполнении запроса к GitHub API.
//     * Завершено - Булево - Признак полного завершения поиска.
//                            Ложь означает, что поиск временно приостановлен и будет продолжен
//                            при следующем вызове метода.
//     * Сообщение - Строка - Диагностическое сообщение.
Функция НайтиРепозитории(ИмяПакета) Экспорт
```
