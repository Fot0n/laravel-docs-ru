# Laravel 9 · HTML-шаблоны

- [Введение](#introduction)
- [Создание и отрисовка шаблонов](#creating-and-rendering-views)
    - [Вложенные каталоги шаблонов](#nested-view-directories)
    - [Использование первого доступного шаблона](#creating-the-first-available-view)
    - [Определение наличия шаблона](#determining-if-a-view-exists)
- [Передача данных шаблону](#passing-data-to-views)
    - [Общедоступные данные для всех шаблонов](#sharing-data-with-all-views)
- [Компоновщики шаблонов](#view-composers)
    - [Создатели шаблонов](#view-creators)
- [Оптимизация шаблонов](#optimizing-views)

<a name="introduction"></a>
## Введение

Конечно, нецелесообразно возвращать целые строки HTML-документов непосредственно из ваших маршрутов и контроллеров. К счастью, шаблоны предоставляют удобный способ разместить весь наш HTML в отдельных файлах. Шаблоны отделяют логику контроллера / приложения от логики представления и хранятся в каталоге `resources/views`. Простой шаблон может выглядеть примерно так:

```blade
<!-- Шаблон сохранен в `resources/views/greeting.blade.php` -->

<html>
    <body>
        <h1>Привет, {{ $name }}</h1>
    </body>
</html>
```

Поскольку этот шаблон сохранен в `resources/views/greeting.blade.php`, мы можем вернуть его, используя глобальный помощник `view`, например:

    Route::get('/', function () {
        return view('greeting', ['name' => 'James']);
    });

> **Примечание**\
> Ищете дополнительную информацию о том, как писать шаблоны Blade? Ознакомьтесь с полной [документацией по Blade](blade.md), чтобы начать работу.

<a name="creating-and-rendering-views"></a>
## Создание и отрисовка шаблонов

Вы можете создать шаблон, поместив файл с расширением `.blade.php` в каталог `resources/views` вашего приложения. Расширение `.blade.php` сообщает фреймворку, что файл содержит [шаблон Blade](blade.md). Шаблоны Blade содержат HTML, а также директивы Blade, которые позволяют легко выводить значения, создавать операторы «если», выполнять итерацию данных и многое другое.

После того, как вы создали шаблон, вы можете вернуть его из маршрута или контроллера вашего приложения, используя глобальный помощник `view`:

    Route::get('/', function () {
        return view('greeting', ['name' => 'James']);
    });

Шаблон также могут быть возвращены с помощью фасада `View`:

    use Illuminate\Support\Facades\View;

    return View::make('greeting', ['name' => 'James']);

Как видно, первый аргумент, переданный помощнику `view`, соответствует имени файла шаблона в каталоге `resources/views`. Второй аргумент – это массив данных, которые должны быть доступны в шаблоне. В этом случае, мы передаем переменную `name`, которая будет выведена в шаблоне с использованием [синтаксиса Blade](blade.md).

<a name="nested-view-directories"></a>
### Вложенные каталоги шаблонов

Шаблоны также могут быть вложены в подкаталоги каталога `resources/views`. «Точечная нотация» используется для указания вложенности шаблона. Например, если ваш шаблон хранится в `resources/views/admin/profile.blade.php`, то вы можете вернуть его из маршрута / контроллера вашего приложения следующим образом:

    return view('admin.profile', $data);

> **Предупреждение**\
> Имена каталогов шаблонов не должны содержать символа `.`.

<a name="creating-the-first-available-view"></a>
### Использование первого доступного шаблона

Используя метод `first` фасада `View`, вы можете отобразить первый шаблон, который существует в переданном массиве шаблонов. Это может быть полезно, если ваше приложение или пакет позволяют настраивать или перезаписывать шаблоны:

    use Illuminate\Support\Facades\View;

    return View::first(['custom.admin', 'admin'], $data);

<a name="determining-if-a-view-exists"></a>
### Определение наличия шаблона

Если вам нужно определить, существует ли шаблон, вы можете использовать фасад `View`. Метод `exists` вернет `true`, если он существует:

    use Illuminate\Support\Facades\View;

    if (View::exists('emails.customer')) {
        //
    }

<a name="passing-data-to-views"></a>
## Передача данных шаблону

Как вы видели в предыдущих примерах, вы можете передать массив данных шаблонам, чтобы сделать эти данные доступными для них:

    return view('greetings', ['name' => 'Victoria']);

При передаче информации таким образом данные должны быть массивом с парами ключ / значение. После предоставления данных в шаблон вы можете получить доступ к каждому значению, используя ключи данных, схожее с `<?php echo $name; ?>`.

В качестве альтернативы передаче полного массива данных вспомогательной функции `view` вы можете использовать метод `with` для добавления некоторых данных в шаблон. Метод `with` возвращает экземпляр объекта представления, так что вы можете продолжить связывание методов перед возвратом шаблона:

    return view('greeting')
                ->with('name', 'Victoria')
                ->with('occupation', 'Astronaut');

<a name="sharing-data-with-all-views"></a>
### Общедоступные данные для всех шаблонов

Иногда требуется сделать данные общедоступными для всех шаблонов, отображаемыми вашим приложением. Вы можете сделать это, используя метод `share` фасада `View`. Как правило, вызов метода `share` осуществляется в методе `boot` поставщика `App\Providers\AppServiceProvider` или вы можете создать отдельного поставщика для их размещения:

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\View;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Регистрация любых служб приложения.
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * Загрузка любых служб приложения.
         *
         * @return void
         */
        public function boot()
        {
            View::share('key', 'value');
        }
    }

<a name="view-composers"></a>
## Компоновщики шаблонов

Компоновщики шаблонов – это замыкания или методы класса, которые вызываются при отрисовки шаблонов. Если у вас есть данные, которые вы хотите привязать к шаблону каждый раз при его отрисовки, компоновщик шаблонов поможет вам организовать эту логику в одном месте. Компоновщики шаблонов особенно полезны, если один и тот же шаблон возвращается несколькими маршрутами или контроллерами в вашем приложении и всегда требует определенного фрагмента данных.

Как правило, компоновщики шаблонов регистрируются в одном из [поставщиков служб](provider.md) вашего приложения. В этом примере мы предположим, что мы создали новый `App\Providers\ViewServiceProvider` для размещения этой логики.

Мы будем использовать метод `composer` фасада `View`, чтобы зарегистрировать компоновщик. Laravel по умолчанию не содержит каталог для классов компоновщиков, поэтому вы можете организовать их, как хотите. Например, вы можете создать каталог `app/View/Composers` для размещения всех компоновщиков вашего приложения:

    <?php

    namespace App\Providers;

    use App\View\Composers\ProfileComposer;
    use Illuminate\Support\Facades\View;
    use Illuminate\Support\ServiceProvider;

    class ViewServiceProvider extends ServiceProvider
    {
        /**
         * Регистрация любых служб приложения.
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * Загрузка любых служб приложения.
         *
         * @return void
         */
        public function boot()
        {
            // Использование компоновщиков на основе классов ...
            View::composer('profile', ProfileComposer::class);

            // Использование анонимных компоновщиков ...
            View::composer('dashboard', function ($view) {
                //
            });
        }
    }

> **Предупреждение**\
> Помните, что если вы создаете нового поставщика служб, который будет содержать регистрации вашего компоновщика, то вам нужно будет добавить поставщика служб в массив `providers` в конфигурационном файле `config/app.php`.

Теперь, когда мы зарегистрировали компоновщик, метод `compose` класса `App\View\Composers\ProfileComposer` будет выполняться каждый раз, когда отрисовывается шаблон профиля. Давайте посмотрим на пример класса компоновщика:

    <?php

    namespace App\View\Composers;

    use App\Repositories\UserRepository;
    use Illuminate\View\View;

    class ProfileComposer
    {
        /**
         * Реализация репозитория User.
         *
         * @var \App\Repositories\UserRepository
         */
        protected $users;

        /**
         * Создать нового компоновщика профиля.
         *
         * @param  \App\Repositories\UserRepository  $users
         * @return void
         */
        public function __construct(UserRepository $users)
        {
            $this->users = $users;
        }

        /**
         * Привязать данные к шаблону.
         *
         * @param  \Illuminate\View\View  $view
         * @return void
         */
        public function compose(View $view)
        {
            $view->with('count', $this->users->count());
        }
    }

Как видите, все компоновщики внедряются через [контейнер служб](container.md), поэтому вы можете указать любые зависимости, которые вам нужны, в конструкторе компоновщика.

<a name="attaching-a-composer-to-multiple-views"></a>
#### Связывание компоновщика с несколькими шаблонами

Вы можете связать компоновщика с несколькими шаблонами одновременно, передав массив шаблонов в качестве первого аргумента методу `composer`:

    use App\View\Composers\MultiComposer;

    View::composer(
        ['profile', 'dashboard'],
        MultiComposer::class
    );

Допускается использование метасимвола подстановки `*`, что позволит вам прикрепить компоновщик ко всем шаблонам:

    View::composer('*', function ($view) {
        //
    });

<a name="view-creators"></a>
### Создатели шаблонов

«Создатели» шаблонов очень похожи на компоновщиков; но, они выполняются сразу после создания экземпляра, а не ожидают отрисовки шаблона. Чтобы зарегистрировать создателя шаблона, используйте метод `creator`:

    use App\View\Creators\ProfileCreator;
    use Illuminate\Support\Facades\View;

    View::creator('profile', ProfileCreator::class);

<a name="optimizing-views"></a>
## Оптимизация шаблонов

По умолчанию шаблоны Blade компилируются по требованию. Когда выполняется запрос, который отрисовывает шаблон, Laravel определит, существует ли скомпилированная версия шаблона. Если файл существует, Laravel далее определит, был ли исходный шаблон изменен позднее скомпилированного. Если скомпилированного шаблона либо не существует, либо исходный шаблон был изменен, Laravel перекомпилирует шаблон.

Компиляция шаблонов во время запроса отрицательно влияет на производительность, поэтому Laravel содержит команду Artisan `view:cache` для предварительной компиляции всех шаблонов, используемых вашим приложением. Для повышения производительности вы можете выполнить эту команду как часть процесса развертывания:

```shell
php artisan view:cache
```

Вы можете использовать команду `view:clear` для очистки кеша шаблонов:

```shell
php artisan view:clear
```
