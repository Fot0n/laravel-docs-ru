# Laravel 9 · База данных · Постраничная навигация

- [Введение](#introduction)
- [Основы использования](#basic-usage)
    - [Пагинация результатов построителя запросов](#paginating-query-builder-results)
    - [Пагинация результатов Eloquent](#paginating-eloquent-results)
    - [Курсорная пагинация](#cursor-pagination)
    - [Самостоятельное создание пагинатора](#manually-creating-a-paginator)
    - [Настройка URL-адресов постраничной навигации](#customizing-pagination-urls)
- [Отображение результатов постраничной навигации](#displaying-pagination-results)
    - [Регулирование количества отображаемых ссылок](#adjusting-the-pagination-link-window)
    - [Преобразование результатов в JSON](#converting-results-to-json)
- [Настройка вида пагинации](#customizing-the-pagination-view)
    - [Использование Bootstrap](#using-bootstrap)
- [Методы экземпляров Paginator и LengthAwarePaginator](#paginator-instance-methods)
- [Методы экземпляра CursorPaginator](#cursor-paginator-instance-methods)

<a name="introduction"></a>
## Введение

В других фреймворках постраничная навигация может быть очень болезненной. Мы надеемся, что подход Laravel к постраничной разбивке станет глотком свежего воздуха. Пагинатор Laravel интегрирован с [построителем запросов](queries.md) и [Eloquent ORM](eloquent.md) и обеспечивает удобную, простую в использовании разбивку на страницы записей базы данных с нулевой конфигурацией.

По умолчанию HTML, генерируемый пагинатором, совместим с [фреймворком Tailwind CSS](https://tailwindcss.com/); однако, также доступна поддержка разбивки на страницы с использованием Bootstrap.

<a name="tailwind-jit"></a>
#### Tailwind JIT

Если вы используете шаблоны разбивки на страницы Tailwind от Laravel и механизм Tailwind JIT, то вы должны убедиться, что ключ содержимого файла `tailwind.config.js` вашего приложения ссылается на шаблоны Laravel, чтобы классы Tailwind не удалялись:

```js
content: [
    './resources/**/*.blade.php',
    './resources/**/*.js',
    './resources/**/*.vue',
    './vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php',
],
```

<a name="basic-usage"></a>
## Основы использования

<a name="paginating-query-builder-results"></a>
### Пагинация результатов построителя запросов

Есть несколько способов разбить элементы на страницы. Самый простой – использовать метод `paginate` [построителя запросов](queries.md)  или в [запросе Eloquent](eloquent.md). Метод `paginate` автоматически устанавливает «предел» и «смещение» в запросе на основе текущей страницы, просматриваемой пользователем. По умолчанию текущая страница определяется значением аргумента `page` строки HTTP-запроса. Это значение автоматически определяется Laravel, а также автоматически вставляется в ссылки, генерируемые пагинатором.

В этом примере единственный аргумент, переданный методу `paginate` – это количество элементов, которые вы хотите отображать «на каждой странице». В этом случае давайте укажем, что мы хотели бы отображать `15` элементов на странице:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\DB;

    class UserController extends Controller
    {
        /**
         * Показать всех пользователей приложения.
         *
         * @return \Illuminate\Http\Response
         */
        public function index()
        {
            return view('user.index', [
                'users' => DB::table('users')->paginate(15)
            ]);
        }
    }

<a name="simple-pagination"></a>
#### Простая пагинация

Метод `paginate` подсчитывает общее количество записей, соответствующих запросу, перед извлечением записей из базы данных. Это сделано для того, чтобы пагинатор знал, сколько всего страниц с записями необходимо сформировать. Однако, если вы не планируете отображать общее количество страниц в пользовательском интерфейсе вашего приложения, запрос количества записей не нужен.

Следовательно, если вам нужно отображать только простые ссылки «Далее» и «Назад» в пользовательском интерфейсе вашего приложения, вы можете использовать метод `simplePaginate` для выполнения одного рационального запроса:

    $users = DB::table('users')->simplePaginate(15);

<a name="paginating-eloquent-results"></a>
### Пагинация результатов Eloquent

Вы также можете разбивать запросы [Eloquent](eloquent.md) на страницы. В этом примере мы разобьем модель `App\Models\User` на страницы и укажем, что мы планируем отображать 15 записей на странице. Как видите, синтаксис почти идентичен разбивке на страницы результатов построителя запросов:

    use App\Models\User;

    $users = User::paginate(15);

Конечно, вы можете вызвать метод `paginate` после указания других ограничений для запроса, таких как выражения `where`:

    $users = User::where('votes', '>', 100)->paginate(15);

Вы также можете использовать метод `simplePaginate` при пагинации моделей Eloquent:

    $users = User::where('votes', '>', 100)->simplePaginate(15);

Точно так же вы можете использовать метод `cursorPaginate` при курсорной пагинации моделей Eloquent:

    $users = User::where('votes', '>', 100)->cursorPaginate(15);

<a name="multiple-paginator-instances-per-page"></a>
#### Несколько экземпляров пагинатора на странице

Иногда требуется использование двух отдельных пагинаторов на одном экране, отображаемом вашим приложением. Однако, если оба экземпляра используют параметр `page` строки запроса для сохранения текущей страницы, то два пагинатора будут конфликтовать. Чтобы разрешить этот конфликт, вы можете передать имя параметра строки запроса, который вы хотите использовать для хранения текущей страницы пагинатора, через третий аргумент, передаваемый методам `paginate`, `simplePaginate` и `cursorPaginate`:

    use App\Models\User;

    $users = User::where('votes', '>', 100)->paginate(
        $perPage = 15, $columns = ['*'], $pageName = 'users'
    );

<a name="cursor-pagination"></a>
### Курсорная пагинация

В то время как `paginate` и `simplePaginate` создают запросы с использованием выражения `offset` SQL, курсорная пагинация работает путем создания выражений `where`, которые сравнивают значения упорядоченных столбцов запроса, обеспечивая наиболее эффективную производительность базы данных среди всех возможных методов пагинации в Laravel. Этот метод пагинации особенно хорошо подходит для больших наборов данных и пользовательских интерфейсов с «бесконечной» прокруткой.

В отличие от пагинации на основе смещения, которая включает номер страницы в строке запроса URL-адресов, сгенерированных пагинатором, курсорная пагинация помещает строку «курсора» в строку запроса. Курсор представляет собой закодированную строку, содержащую текущее положение, с которого следующий запрос пагинации должен начать постраничную разбивку, и направление, в котором она должна выполняться:

```nothing
http://localhost/users?cursor=eyJpZCI6MTUsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX0
```

Вы можете создать экземпляр пагинатора на основе курсора с помощью метода `cursorPaginate` построителя запросов. Этот метод возвращает экземпляр `Illuminate\Pagination\CursorPaginator`:

    $users = DB::table('users')->orderBy('id')->cursorPaginate(15);

После того, как вы получили экземпляр курсора, вы можете [отобразить результаты постраничной навигации](#displaying-pagination-results), как и обычно при использовании методов `paginate` и `simplePaginate`. Для получения дополнительной информации о методах экземпляра, предлагаемых  пагинатором на основе курсора, обратитесь к [документации по методам экземпляра `CursorPaginator`](#cursor-paginator-instance-methods).

> **Предупреждение**\
> Ваш запрос должен содержать выражение `order by`, чтобы можно было использовать курсорную пагинацию.

<a name="cursor-vs-offset-pagination"></a>
#### Курсорная пагинация против пагинации на основе смещения

Чтобы проиллюстрировать различия между этими постраничными разбивками, давайте рассмотрим несколько примеров запросов SQL. Оба следующих запроса будут отображать «вторую страницу» результатов для таблицы `users`, упорядоченных по `id`:

```sql
# Пагинация на основе смещения ...
select * from users order by id asc limit 15 offset 15;

# Курсорная пагинация ...
select * from users where id > 15 order by id asc limit 15;
```

Запрос курсорной разбивки предлагает следующие преимущества по сравнению со смещенной разбивкой:

- Для больших наборов данных курсорная разбивка будет обеспечивать лучшую производительность, если столбцы, по которым выполняется упорядочение (`order by`) индексируются. Это связано с тем, что выражение `offset` сканирует все ранее сопоставленные данные.
- Для наборов данных с частой записью при разбивке на основе смещения могут быть пропущены записи или показаны дубликаты, если результаты были недавно добавлены / удалены со страницы, которую пользователь просматривает в данный момент.

Однако курсорная пагинация имеет следующие ограничения:

- Как и `simplePaginate`, курсорная пагинация может использоваться только для отображения ссылок «Далее» и «Назад» и не поддерживает создание ссылок с номерами страниц.
- Требуется, чтобы порядок был основан как минимум на одном уникальном столбце или уникальной комбинации столбцов. Столбцы со значениями `null` не поддерживаются.
- Выражения запроса в конструкциях `order by` поддерживаются только в том случае, если они имеют псевдонимы и добавлены в конструкцию `select`.

<a name="manually-creating-a-paginator"></a>
### Самостоятельное создание пагинатора

По желанию можно вручную создать экземпляр пагинатора, передав ему массив элементов, которые у вас уже есть в памяти. Вы можете сделать это, создав экземпляр `Illuminate\Pagination\Paginator`, `Illuminate\Pagination\LengthAwarePaginator` или `Illuminate\Pagination\CursorPaginator`, в зависимости от ваших потребностей.

Классам `Paginator` и `CursorPaginator` не требуется знать общее количество элементов в результирующем наборе; однако из-за этого у классов нет методов для получения индекса последней страницы. `LengthAwarePaginator` принимает почти те же аргументы, что и `Paginator`; однако, для этого требуется подсчет общего количества элементов в результирующем наборе.

Другими словами, `Paginator` соответствует методу `simplePaginate` построителя запросов, `CursorPaginator` – методу `cursorPaginate`, а `LengthAwarePaginator` – методу `paginate`.

> **Предупреждение**\
> При ручном создании экземпляра пагинатора вы должны самостоятельно «разрезать» массив результатов, который вы передаете в пагинатор. Если вы не знаете, как это сделать, ознакомьтесь с функцией PHP [`array_slice`](https://www.php.net/manual/ru/function.array-slice.php).

<a name="customizing-pagination-urls"></a>
### Настройка URL-адресов постраничной навигации

По умолчанию ссылки, созданные пагинатором, будут соответствовать URI текущего запроса. Однако метод `withPath` пагинатора позволяет вам скорректировать URI, используемый пагинатором при генерации ссылок. Например, если вы хотите, чтобы пагинатор генерировал ссылки типа `http://example.com/admin/users?page=N`, вы должны передать `/admin/users` `withPath`:

    use App\Models\User;

    Route::get('/users', function () {
        $users = User::paginate(15);

        $users->withPath('/admin/users');

        //
    });

<a name="appending-query-string-values"></a>
#### Добавление значений в строку запроса

Вы можете добавить параметр в строку запроса навигационных ссылок с помощью метода `appends`. Например, чтобы добавить `sort=votes` к каждой ссылке пагинации, вы должны сделать следующий вызов `appends`:

    use App\Models\User;

    Route::get('/users', function () {
        $users = User::paginate(15);

        $users->appends(['sort' => 'votes']);

        //
    });

Вы можете использовать метод `withQueryString`, если хотите добавить все значения строки текущего запроса к ссылкам постраничной навигации:

    $users = User::paginate(15)->withQueryString();

<a name="appending-hash-fragments"></a>
#### Добавление фрагментов хеша

Если вам нужно добавить «хеш-фрагмент» к URL-адресам, сгенерированным пагинатором, вы можете использовать метод `fragment`. Например, чтобы добавить `#users` в конец каждой навигационной ссылки, вы должны вызвать метод `fragment` следующим образом:

    $users = User::paginate(15)->fragment('users');

<a name="displaying-pagination-results"></a>
## Отображение результатов постраничной навигации

При вызове метода `paginate` вы получите экземпляр `Illuminate\Pagination\LengthAwarePaginator`. При вызове метода `simplePaginate` вы получите экземпляр `Illuminate\Pagination\Paginator`. И, наконец, вызов метода `cursorPaginate` возвращает экземпляр `Illuminate\Pagination\CursorPaginator`.

Эти объекты содержат несколько методов, описывающих результирующий набор. В дополнение к этим вспомогательным методам, экземпляры пагинатора являются итераторами и могут быть перебраны как массив. Итак, как только вы получили результаты, вы можете отобразить результаты и отрисовать ссылки на страницы, используя [Blade](blade.md):

```blade
<div class="container">
    @foreach ($users as $user)
        {{ $user->name }}
    @endforeach
</div>

{{ $users->links() }}
```

Метод `links` отрисует ссылки на остальные страницы в результирующем наборе. Каждая из этих ссылок уже будет содержать соответствующую строковую переменную запроса `page`. Помните, что HTML, сгенерированный методом `links`, совместим с [фреймворком Tailwind CSS](https://tailwindcss.com).

<a name="adjusting-the-pagination-link-window"></a>
### Регулирование количества отображаемых ссылок

Пагинатор отображает навигационные ссылки, которые содержат номер текущей страницы, а также ссылки для трех страниц до и после текущей страницы. Используя метод `onEachSide`, вы можете контролировать количество дополнительных ссылок, отображаемых с каждой стороны от текущей страницы в среднем блоке ссылок, созданных пагинатором:

```blade
{{ $users->onEachSide(5)->links() }}
```

<a name="converting-results-to-json"></a>
### Преобразование результатов в JSON

Классы пагинатора Laravel реализуют контракт интерфейса `Illuminate\Contracts\Support\Jsonable` и содержат метод `toJson`, поэтому очень легко преобразовать результаты в JSON. Вы также можете преобразовать экземпляр пагинатора в JSON, вернув его из маршрута или действия контроллера:

    use App\Models\User;

    Route::get('/users', function () {
        return User::paginate();
    });

JSON из пагинатора будет включать метаинформацию, такую как `total`, `current_page`, `last_page` и другие. Записи результатов доступны через ключ `data` в массиве JSON. Вот пример JSON, созданного путем возврата экземпляра пагинатора из маршрута:

    {
       "total": 50,
       "per_page": 15,
       "current_page": 1,
       "last_page": 4,
       "first_page_url": "http://laravel.app?page=1",
       "last_page_url": "http://laravel.app?page=4",
       "next_page_url": "http://laravel.app?page=2",
       "prev_page_url": null,
       "path": "http://laravel.app",
       "from": 1,
       "to": 15,
       "data":[
            {
                // Запись ...
            },
            {
                // Запись ...
            }
       ]
    }

<a name="customizing-the-pagination-view"></a>
## Настройка вида пагинации

По умолчанию сгенерированные шаблоны для отображения навигационных ссылок, совместимы со структурой [фреймворка Tailwind CSS](https://tailwindcss.com). Однако, если вы не используете Tailwind, вы можете определять свои собственные шаблоны для отображения этих ссылок. При вызове метода `links` в экземпляре пагинатора вы можете передать имя шаблона в качестве первого аргумента метода:

```blade
{{ $paginator->links('view.name') }}

// Передача дополнительных данных в шаблон ...
{{ $paginator->links('view.name', ['foo' => 'bar']) }}
```

Однако, самый простой способ отредактировать шаблоны постраничной навигации – это экспортировать их в каталог `resources/views/vendor` с помощью команды `vendor:publish`:

```shell
php artisan vendor:publish --tag=laravel-pagination
```

Эта команда поместит шаблоны в каталог `resources/views/vendor/pagination` вашего приложения. Файл `tailwind.blade.php` в этом каталоге соответствует шаблону постраничной навигации по умолчанию. Вы можете отредактировать этот файл для изменения HTML-кода навигации.

Если вы хотите назначить другой файл в качестве шаблона постраничной навигации по умолчанию, то вы можете вызвать методы `defaultView` и `defaultSimpleView` пагинатора в методе `boot` поставщика `App\Providers\AppServiceProvider`:

    <?php

    namespace App\Providers;

    use Illuminate\Pagination\Paginator;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Загрузка любых служб приложения.
         *
         * @return void
         */
        public function boot()
        {
            Paginator::defaultView('view-name');

            Paginator::defaultSimpleView('view-name');
        }
    }

<a name="using-bootstrap"></a>
### Использование Bootstrap

Laravel содержит шаблоны постраничной навигации, созданные с использованием [Bootstrap CSS](https://getbootstrap.com/). Чтобы использовать эти шаблоны вместо шаблонов Tailwind по умолчанию, вы можете вызвать метод пагинатора `useBootstrapFour` или `useBootstrapFive` в методе `boot` поставщика `App\Providers\AppServiceProvider`:

    use Illuminate\Pagination\Paginator;

    /**
     * Загрузка любых служб приложения.
     *
     * @return void
     */
    public function boot()
    {
        Paginator::useBootstrapFive();
        Paginator::useBootstrapFour();
    }

<a name="paginator-instance-methods"></a>
## Методы экземпляров Paginator и LengthAwarePaginator

Каждый экземпляр предоставляет дополнительную информацию о постраничной навигации с помощью следующих методов:

Метод  |  Описание
-------  |  -----------
`$paginator->count()`  |  Получить количество элементов для текущей страницы.
`$paginator->currentPage()`  |  Получить номер текущей страницы.
`$paginator->firstItem()`  |  Получить номер первого элемента в результатах.
`$paginator->getOptions()`  |  Получить параметры пагинатора.
`$paginator->getUrlRange($start, $end)`  |  Создать диапазон URL-адресов для пагинации.
`$paginator->hasPages()`  |  Определить, достаточно ли элементов для разделения на несколько страниц.
`$paginator->hasMorePages()`  |  Определить, есть ли еще элементы в хранилище данных.
`$paginator->items()`  |  Получить элементы для текущей страницы.
`$paginator->lastItem()`  |  Получить номер последнего элемента в результатах.
`$paginator->lastPage()`  |  Получить номер последней доступной страницы. (Недоступно при использовании `simplePaginate`).
`$paginator->nextPageUrl()`  |  Получить URL-адрес следующей страницы.
`$paginator->onFirstPage()`  |  Определить, находится ли пагинатор на первой странице.
`$paginator->perPage()`  |  Количество элементов, отображаемых на каждой странице.
`$paginator->previousPageUrl()`  |  Получить URL-адрес предыдущей страницы.
`$paginator->total()`  |  Определить общее количество элементов запроса в хранилище данных. (Недоступно при использовании `simplePaginate`).
`$paginator->url($page)`  |  Получить URL-адрес для конкретного номера страницы.
`$paginator->getPageName()`  |  Получить переменную строки запроса, используемую для хранения страницы.
`$paginator->setPageName($name)`  |  Установить переменную строки запроса, используемую для хранения страницы.

<a name="cursor-paginator-instance-methods"></a>
## Методы экземпляра CursorPaginator

Каждый экземпляр `CursorPaginator` предоставляет дополнительную информацию о постраничной навигации с помощью следующих методов:

Метод  |  Описание
-------  |  -----------
`$paginator->count()`  |  Получить количество элементов для текущей страницы.
`$paginator->cursor()`  |  Получить текущий экземпляр курсора.
`$paginator->getOptions()`  |  Получить параметры пагинатора.
`$paginator->hasPages()`  |  Определить, достаточно ли элементов для разделения на несколько страниц.
`$paginator->hasMorePages()`  |  Определить, есть ли еще элементы в хранилище данных.
`$paginator->getCursorName()`  |  Получить переменную строки запроса, используемую для хранения курсора.
`$paginator->items()`  |  Получить элементы для текущей страницы.
`$paginator->nextCursor()`  |  Получить экземпляр курсора для следующего набора элементов.
`$paginator->nextPageUrl()`  |  Получить URL-адрес следующей страницы.
`$paginator->onFirstPage()`  |  Определить, находится ли пагинатор на первой странице.
`$paginator->onLastPage()`  |  Определить, находится ли пагинатор на последней странице.
`$paginator->perPage()`  |  Количество элементов, отображаемых на каждой странице.
`$paginator->previousCursor()`  |  Получить экземпляр курсора для предыдущего набора элементов.
`$paginator->previousPageUrl()`  |  Получить URL-адрес предыдущей страницы.
`$paginator->setCursorName()`  |  Задать переменную строки запроса, используемую для хранения курсора.
`$paginator->url($cursor)`  |  Получить URL-адрес для данного экземпляра курсора.
