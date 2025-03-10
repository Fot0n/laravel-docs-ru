# Laravel 9 · Подтверждение адреса электронной почты

- [Введение](#introduction)
    - [Подготовка модели](#model-preparation)
    - [Подготовка базы данных](#database-preparation)
- [Маршрутизация](#verification-routing)
    - [Уведомление о подтверждении электронной почты](#the-email-verification-notice)
    - [Обработчик проверки электронной почты](#the-email-verification-handler)
    - [Повторная отправка письма с подтверждением](#resending-the-verification-email)
    - [Защита маршрутов](#protecting-routes)
- [Настройка](#customization)
- [События](#events)

<a name="introduction"></a>
## Введение

Многие веб-приложения требуют от пользователей подтверждения своего адреса электронной почты перед использованием приложения. Вместо того, чтобы заставлять вас самостоятельно реализовывать этот функционал повторно для каждого создаваемого вами приложения, Laravel предлагает удобные встроенные службы для отправки и проверки запросов подтверждения адреса электронной почты.

> **Примечание**\
> Хотите быстро начать? Установите один из [стартовых комплектов](starter-kits.md) в новое приложение Laravel. Стартовые комплекты позаботятся о построении всей вашей системы аутентификации, включая поддержку подтверждения электронной почты.

<a name="model-preparation"></a>
### Подготовка модели

Убедитесь, что ваша модель `App\Models\User` реализует контракт `Illuminate\Contracts\Auth\MustVerifyEmail`:

    <?php

    namespace App\Models;

    use Illuminate\Contracts\Auth\MustVerifyEmail;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable implements MustVerifyEmail
    {
        use Notifiable;

        // ...
    }

Как только этот интерфейс будет добавлен в вашу модель, вновь зарегистрированным пользователям будет автоматически отправлено электронное письмо со ссылкой для подтверждения адреса электронной почты. Изучив `App\Providers\EventServiceProvider`, вы можете увидеть, что Laravel уже содержит [слушатель](events.md) `SendEmailVerificationNotification`, который прикреплен к событию `Illuminate\Auth\Events\Registered`. Этот слушатель события отправит по электронной почте пользователю ссылку для подтверждения.

Если вы самостоятельно выполняете регистрацию в своем приложении вместо использования [стартового комплекта](starter-kits.md), то вы должны убедиться, что запускаете событие `Illuminate\Auth\Events\Registered` после успешной регистрации пользователя:

    use Illuminate\Auth\Events\Registered;

    event(new Registered($user));

<a name="database-preparation"></a>
### Подготовка базы данных

Ваша таблица `users` должна содержать столбец `email_verified_at` для сохранения даты и времени подтверждения адреса электронной почты пользователем. По умолчанию миграция таблицы пользователей, содержащаяся в Laravel, уже содержит этот столбец. Просто запустите миграцию базы данных:

```shell
php artisan migrate
```

<a name="verification-routing"></a>
## Маршрутизация

Чтобы правильно реализовать подтверждение электронной почты, необходимо определить три маршрута. Во-первых, потребуется маршрут для отображения уведомления пользователю о том, что он должен щелкнуть ссылку подтверждения электронной почты в письме, которое Laravel отправит ему после регистрации.

Во-вторых, потребуется маршрут для обработки запросов, сгенерированных, когда пользователь щелкает ссылку подтверждения электронной почты в электронном письме.

В-третьих, потребуется маршрут для повторной отправки ссылки для подтверждения, если пользователь случайно потеряет первую ссылку для подтверждения.

<a name="the-email-verification-notice"></a>
### Уведомление о подтверждении электронной почты

Как упоминалось ранее, должен быть определен маршрут, который будет возвращать страницу, указывающую пользователю щелкнуть ссылку для подтверждения электронной почты, которая была отправлена ему Laravel по электронной почте после регистрации. Эта страница будет отображаться для пользователей, когда они попытаются получить доступ к другим частям приложения без предварительной проверки своего адреса электронной почты. Помните, что ссылка автоматически отправляется пользователю по электронной почте, если ваша модель `App\Models\User` реализует интерфейс `MustVerifyEmail`:

    Route::get('/email/verify', function () {
        return view('auth.verify-email');
    })->middleware('auth')->name('verification.notice');

Маршрут, который возвращает уведомление о подтверждении по электронной почте, должен называться `verification.notice`. Важно, чтобы маршруту было присвоено это точное имя, поскольку посредник `verify`, [включенный в Laravel](#protecting-routes), будет автоматически перенаправлять на это имя маршрута, если пользователь не подтвердил свой адрес электронной почты.

> **Примечание**\
> При выполнении проверки электронной почты самостоятельно, вам необходимо определить содержание страницы уведомления о проверке. Если вам необходим каркас, включающий все необходимые страницы для аутентификации и проверки, ознакомьтесь со [стартовыми комплектами приложений Laravel](starter-kits.md).

<a name="the-email-verification-handler"></a>
### Обработчик проверки электронной почты

Затем, нам нужно определить маршрут, который будет обрабатывать запросы, сгенерированные, когда пользователь щелкает ссылку подтверждения электронной почты, которая была отправлена ему по электронной почте. Этот маршрут должен называться `verification.verify` и ему должны быть назначены посредники `auth` и `signed`:

    use Illuminate\Foundation\Auth\EmailVerificationRequest;

    Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {
        $request->fulfill();

        return redirect('/home');
    })->middleware(['auth', 'signed'])->name('verification.verify');

Прежде чем двигаться дальше, давайте подробнее рассмотрим этот маршрут. Во-первых, вы заметите, что мы используем тип запроса `EmailVerificationRequest` вместо типичного экземпляра `Illuminate\Http\Request`. `EmailVerificationRequest` – это [запрос формы](validation.md#form-request-validation), который включен в Laravel. Этот запрос автоматически позаботится о проверке параметров запроса `id` и `hash`.

Далее, мы можем приступить непосредственно к вызову метода `fulfill` запроса. Этот метод вызовет метод `markEmailAsVerified` для аутентифицированного пользователя и запустит событие `Illuminate\Auth\Events\Verified`. Метод `markEmailAsVerified` доступен для модели по умолчанию `App\Models\User` через базовый класс `Illuminate\Foundation\Auth\User`. После подтверждения адреса электронной почты пользователя вы можете перенаправить его куда пожелаете.

<a name="resending-the-verification-email"></a>
### Повторная отправка письма с подтверждением

Иногда пользователь может потерять или случайно удалить письмо с подтверждением адреса электронной почты. Чтобы учесть это, вы можете определить маршрут, позволяющий пользователю запрашивать повторную отправку письма с подтверждением. Затем, вы можете сделать запрос по этому маршруту, поместив простую кнопку отправки формы на [странице уведомления о подтверждении](#the-email-verification-notice):

    use Illuminate\Http\Request;

    Route::post('/email/verification-notification', function (Request $request) {
        $request->user()->sendEmailVerificationNotification();

        return back()->with('message', 'Verification link sent!');
    })->middleware(['auth', 'throttle:6,1'])->name('verification.send');

<a name="protecting-routes"></a>
### Защита маршрутов

[Посредник маршрута](middleware.md) может использоваться только для того, чтобы разрешить доступ к конкретному маршруту только подтвержденным пользователям. Laravel содержит посредник `verified`, который ссылается на класс `Illuminate\Auth\Middleware\EnsureEmailIsVerified`. Поскольку этот посредник уже зарегистрирован в HTTP-ядре вашего приложения, все, что вам нужно сделать, так это назначить посредник маршруту:

    Route::get('/profile', function () {
        // Только подтвержденные пользователи могут получить доступ к этому маршруту ...
    })->middleware('verified');

Если непроверенный пользователь попытается получить доступ к маршруту, которому назначен этот посредник, то он будет автоматически перенаправлен на [именованный маршрут](routing.md#named-routes) `verification.notice`.

<a name="customization"></a>
## Настройка

<a name="verification-email-customization"></a>
#### Настройка подтверждения адреса электронной почты

Хотя уведомление о подтверждении электронной почты по умолчанию должно удовлетворять требованиям большинства приложений, Laravel позволяет вам изменить сообщение подтверждения электронной почты.

Для начала, передайте замыкание методу `toMailUsing` уведомления `Illuminate\Auth\Notifications\VerifyEmail`. Замыкание получит экземпляр модели, содержащий уведомление, а также подписанный URL-адрес подтверждения электронной почты, который пользователь должен посетить для проверки адреса электронной почты. Замыкание должно вернуть экземпляр `Illuminate\Notifications\Messages\MailMessage`. Как правило, вызов метода `toMailUsing` осуществляется в методе `boot` поставщика `App\Providers\AuthServiceProvider`:

    use Illuminate\Auth\Notifications\VerifyEmail;
    use Illuminate\Notifications\Messages\MailMessage;

    /**
     * Регистрация любых служб аутентификации / авторизации.
     *
     * @return void
     */
    public function boot()
    {
        // ...

        VerifyEmail::toMailUsing(function ($notifiable, $url) {
            return (new MailMessage)
                ->subject('Verify Email Address')
                ->line('Click the button below to verify your email address.')
                ->action('Verify Email Address', $url);
        });
    }

> **Примечание**\
> Чтобы узнать больше о почтовых уведомлениях, обратитесь к [документации по почтовым уведомлениям](notifications.md#mail-notifications).

<a name="events"></a>
## События

При использовании [стартовых комплектов](starter-kits.md) Laravel запускает [события](events.md) в процессе проверки электронной почты. Если вы самостоятельно обрабатываете проверку электронной почты для своего приложения, то вы должны запускать эти события после завершения проверки. Как правило, регистрация слушателей этого события осуществляется в поставщике `App\Providers\EventServiceProvider`:

    /**
     * Карта слушателей событий приложения.
     *
     * @var array
     */
    protected $listen = [
        'Illuminate\Auth\Events\Verified' => [
            'App\Listeners\LogVerifiedUser',
        ],
    ];
