# BearyChat for Laravel

这个 Laravel 扩展包封装了 [BearyChat PHP 扩展包][1]，用于向 [BearyChat][] 发送 [Incoming Webhook][Webhook] 消息。

该扩展包兼容 [Laravel 5](#laravel-5) 、 [Laravel 4](#laravel-4) 和 [Lumen](#lumen)。

+ :us: [**Documentation in English**](README.md)

## 安装

你可以使用 [Composer][] 安装此扩展包：
```
composer require elfsundae/laravel-bearychat
```
更新完 composer 后，你可以根据以下指引来配置你的 Laravel 应用。

### Laravel 5

将 service provider 添加到 `config/app.php` 中的 `providers` 数组中。
```php
ElfSundae\BearyChat\Laravel\ServiceProvider::class,
```
然后发布 BearyChat 的配置文件：
```shell
$ php artisan vendor:publish --provider="ElfSundae\BearyChat\Laravel\ServiceProvider"
```
编辑配置文件 `config/bearychat.php` ，配置 webhook 和消息预设值。

### Laravel 4

将 service provider 添加到 `config/app.php` 中的 `providers` 数组中。
```php
'ElfSundae\BearyChat\Laravel\ServiceProvider',
```
然后发布 BearyChat 的配置文件：
```shell
$ php artisan config:publish elfsundae/laravel-bearychat
```
编辑配置文件 `app/config/packages/elfsundae/laravel-bearychat/config.php` ，配置 webhook 和消息预设值。

### Lumen

在 `bootstrap/app.php` 中注册 service provider:
```php
$app->register(ElfSundae\BearyChat\Laravel\ServiceProvider::class);
```
然后从扩展包目录拷贝 BearyChat 配置文件到你应用的 `config/bearychat.php`:
```shell
$ cp vendor/elfsundae/laravel-bearychat/src/config/config.php config/bearychat.php
```
为了使配置生效，必须在 `bootstrap/app.php` 中激活：
```php
$app->configure('bearychat');
```
编辑配置文件 `config/bearychat.php` ，配置 webhook 和消息预设值。

如果你想使用 `BearyChat` 门面 (facade)，必须在 `bootstrap/app.php` 文件中取消 `$app->withFacades()` 的代码注释。

## 使用方法

通过 `BearyChat` 门面 (facade) 或者 `bearychat()` 帮助函数，可以得到 BearyChat `Client` 实例。

```php
BearyChat::send('message');

bearychat()->sendTo('@elf', 'Hi!');
```

调用 `BearyChat` 门面的 `client` 方法并传入一个 client 名字，或者将 client 名字传入 `bearychat()` 函数，可以得到其他不同的 BearyChat `Client` 实例。作为参数的 client 名字必须在 BearyChat 的配置文件中事先定义。如果不传 client 名字参数，默认会使用 `"default"` 作为 client 的名字。

```php
BearyChat::client('dev')->send('foo');

bearychat('admin')->send('bar');
```

> **更多高级用法，请参阅 [BearyChat PHP 扩展包的文档][2]。**

### 异步消息

发送一条 BearyChat 消息实际上是向 Incoming Webhook 发送同步 HTTP 请求，所以这在一定程度上会延长应用的响应时间。可以使用 Laravel 强悍的[队列系统][queue system]来异步发送消息。

下面是一个 Laravel 5.2 应用的队列任务的示例：

```php
<?php

namespace App\Jobs;

use App\Jobs\Job;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use ElfSundae\BearyChat\Message;
use Exception;
use Log;

class SendBearyChat extends Job implements ShouldQueue
{
    use SerializesModels, InteractsWithQueue;

    /**
     * The Message instance for sending.
     *
     * @var \ElfSundae\BearyChat\Message
     */
    protected $message;

    /**
     * Create a new job instance.
     */
    public function __construct($message)
    {
        $this->message = $message;
    }

    /**
     * Execute the job.
     */
    public function handle()
    {
        if ($this->message instanceof Message) {
            try {
                $this->message->send();
            } catch (Exception $e) {
                Log::error(
                    'Exception when processing '.get_class($this)." \n".$e,
                    $this->message->toArray()
                );
                $this->release(10);
            }
        } else {
            $this->delete();
        }
    }
}
```

然后在任意包含了 `DispatchesJobs` trait 的类中调用 `dispatch` 方法，或者使用全局的 `dispatch()` 函数，就可以将 `SendBearyChat` 任务派遣到队列中执行。例如：

```php
$order = PayOrder::create($request->all());

dispatch(new \App\Jobs\SendBearyChat(
    bearychat()->text('New order!')
    ->add($order, $order->name, $order->image_url)
));
```

### 报告 Laravel 异常

BearyChat 的一个常见用法是实时报告 Laravel 应用的异常或错误日志。要实现这个功能，只需要重载现有的异常处理类中的 `report` 方法，并添加发送异常信息到 BearyChat ：

```php
/**
 * Report or log an exception.
 *
 * This is a great spot to send exceptions to Sentry, Bugsnag, etc.
 *
 * @param  \Exception  $e
 * @return void
 */
public function report(Exception $e)
{
    if ($this->shouldReport($e)) {
        dispatch(new \App\Jobs\SendBearyChat(
            bearychat('server')->text('New Exception!')
            ->notification('New Exception: '.get_class($e))
            ->add($e, get_class($e))
            ->add([
                'URL' => app('request')->fullUrl(),
                'UserAgent' => app('request')->server('HTTP_USER_AGENT')
            ])
        ));
    }

    parent::report($e);
}
```

## 许可协议

BearyChat Laravel 扩展包在 [MIT 许可协议](LICENSE)下提供和使用。

[1]: https://github.com/ElfSundae/BearyChat
[2]: https://github.com/ElfSundae/BearyChat/blob/master/README_zh.md
[Webhook]: https://bearychat.com/integrations/incoming
[BearyChat]: https://bearychat.com
[Composer]: https://getcomposer.org
[queue system]: https://laravel.com/docs/5.2/queues