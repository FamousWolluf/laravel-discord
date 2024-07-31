# Discord notification channel for Laravel 6.0+

[![Latest Version on Packagist](https://img.shields.io/packagist/v/famouswolluf/laravel-discord.svg?style=flat-square)](https://packagist.org/packages/famouswolluf/laravel-discord)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Build Status](https://img.shields.io/github/workflow/status/famouswolluf/laravel-discord/PHP.svg?style=flat-square)](https://github.com/famouswolluf/laravel-discord/actions)
[![Quality Score](https://img.shields.io/scrutinizer/g/famouswolluf/laravel-discord.svg?style=flat-square)](https://scrutinizer-ci.com/g/famouswolluf/laravel-discord)
[![Code Coverage](https://img.shields.io/scrutinizer/coverage/g/famouswolluf/laravel-discord/master.svg?style=flat-square)](https://scrutinizer-ci.com/g/famouswolluf/laravel-discord/?branch=master)
[![Total Downloads](https://img.shields.io/packagist/dt/famouswolluf/laravel-discord.svg?style=flat-square)](https://packagist.org/packages/famouswolluf/laravel-discord)

This package makes it easy to send notifications using the [Discord bot API](https://discord.com/developers/docs/intro) with Laravel.

## Fork
This is a fork on https://github.com/laravel-notification-channels/discord with extra functionality.

## Added functionalities
Add a (custom) emoji reaction to the message send.

## Contents

- [Discord notification channel for Laravel 6.0+](#discord-notification-channel-for-laravel-56)
    - [Contents](#contents)
    - [Installation](#installation)
        - [Setting up your Discord bot](#setting-up-your-discord-bot)
    - [Usage](#usage)
        - [Available Message methods](#available-message-methods)
    - [Example embed message](#example-embed-message)
    - [Changelog](#changelog)
    - [Testing](#testing)
    - [Security](#security)
    - [Contributing](#contributing)
    - [Credits](#credits)
    - [License](#license)


## Installation

You can install the package via composer:

```bash
composer require famouswolluf/laravel-discord
```

Next, you must load the service provider:

```php
// config/app.php
'providers' => [
    // ...
    NotificationChannels\Discord\DiscordServiceProvider::class,
],
```

### Setting up your Discord bot

1. [Create a Discord application.](https://discord.com/developers/applications)
2. Click the `Create a Bot User` button on your Discord application.
3. Paste your bot's API token, found under `App Bot User`, in your `services.php` config file:

    ```php
    // config/services.php
    'discord' => [
        'token' => 'YOUR_API_TOKEN',
    ],
    ```

4. Add the bot to your server and identify it by running the artisan command:

    ```shell
    php artisan discord:setup
    ```

## Usage

In every model you wish to be notifiable via Discord, you must add a channel ID property to that model accessible through a `routeNotificationForDiscord` method:

```php
class Guild extends Eloquent
{
    use Notifiable;

    public function routeNotificationForDiscord()
    {
        return $this->discord_channel;
    }
}
```

> **NOTE**: Discord handles direct messages as though they are a regular channel. If you wish to allow users to receive direct messages from your bot, you will need to create a private channel with that user.
>
> An example workflow may look like the following:
>
> 1. Your `users` table has two discord columns: `discord_user_id` and `discord_private_channel_id`
> 2. When a user updates their Discord user ID (`discord_user_id`), generate and save a private channel ID (`discord_private_channel_id`)
> 3. Return the user's `discord_private_channel_id` in the `routeNotificationForDiscord` method on the `User` model
>
> You can generate direct message channels by using the `getPrivateChannel` method in the `NotificationChannels\Discord\Discord` class
>
> ```php
> use NotificationChannels\Discord\Discord;
>
> class UserDiscordSettingsController
> {
>     public function store(Request $request)
>     {
>         $userId = $request->input('discord_user_id');
>         $channelId = app(Discord::class)->getPrivateChannel($userId);
>
>         Auth::user()->update([
>             'discord_user_id' => $userId,
>             'discord_private_channel_id' => $channelId,
>         ]);
>     }
> }
> ```
>
> Please take note that the `getPrivateChannel` method only accepts [Discord's snowflake IDs](https://discord.com/developers/docs/reference#snowflakes). There is no API route provided by Discord to lookup a user's ID by their name and tag, and the process for copying and pasting a user ID can be confusing to some users. Because of this, it is recommended to add the option for users to connect their Discord account to their account within your application either by logging in with Discord or linking it to their pre-existing account.

You may now tell Laravel to send notifications to Discord channels in the `via` method:

```php
// ...
use NotificationChannels\Discord\DiscordChannel;
use NotificationChannels\Discord\DiscordMessage;

class GameChallengeNotification extends Notification
{
    public $challenger;

    public $game;

    public function __construct(Guild $challenger, Game $game)
    {
        $this->challenger = $challenger;
        $this->game = $game;
    }

    public function via($notifiable)
    {
        return [DiscordChannel::class];
    }

    public function toDiscord($notifiable)
    {
        return DiscordMessage::create("You have been challenged to a game of *{$this->game->name}* by **{$this->challenger->name}**!");
    }
}
```

### Available Message methods

* `body(string)`: Set the content of the message. ([Supports basic markdown](https://support.discord.com/hc/en-us/articles/210298617-Markdown-Text-101-Chat-Formatting-Bold-Italic-Underline-))
* `embed(array)`: Set the embedded content. ([View embed structure](https://discord.com/developers/docs/resources/channel#embed-object))
* `components(array)`: Set the component content. ([View component structure](https://discord.com/developers/docs/interactions/message-components#component-object))
* * `reaction(string)`: Set a (custom) emoji as reaction. ([View reaction structure](https://discord.com/developers/docs/resources/channel#create-reaction))


## Example Embed Message
```php
// ...
use NotificationChannels\Discord\DiscordChannel;
use NotificationChannels\Discord\DiscordMessage;

class SendDiscordEmbedMessage extends Notification
{
    private $channelId;
    private $messageData;
    private $reaction;

    public function __construct($channelId = null, $messageContent = null, $reaction = null)
    {
        $this->channelId = $channelId;
        $this->messageData = $messageContent;
        $this->reaction = $reaction;
    }

    public function getChannelId()
    {
        return $this->channelId;
    }

    public function via($notifiable)
    {
        return [DiscordChannel::class];
    }


    public function toDiscord($notifiable)
    {
        $discordMessage = DiscordMessage::create()
            ->embed([
                'title'         => $this->messageData['title'],
                'description'   => $this->messageData['message'],
                'color'         => $this->messageData['color'],
                'image'         => [
                    'url'   => $this->messageData['image']
                ],
            ]);

        if ($this->reaction) {
            $discordMessage->reaction($this->reaction);
        }

        return $discordMessage;
    }
}
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information what has changed recently.

## Testing

```bash
$ composer test
```

## Credits

- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [LICENSE](LICENSE.md) for more information.
