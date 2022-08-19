---
title: BGA Internals explained, part n: Notifications &ndash; Translation &ndash; `format_string_recursive` method &ndash; Javascript HTML log injection
author: Thibault HUTTIN-PASSERON
date: 2022-08-18 20:30:00 +0200
categories: [Tips,Documentation]
tags: [notification,log injection,translation]
pin: false
---

# BGA Internals explained, part n
## Notifications &ndash;  Translation &ndash; `format_string_recursive` method &ndash; Javascript HTML log injection

This article is intended to be part of a series explaining some of the internals of the BGA framework. At the time of
writing the first part hasn't been written as I'll be writing these articles on the fly, and ordering them afterwards.
This is why it's titled *part n*.

Let's dive in head first:

### Notifications:

#### What are notifications ?

Notifications are at the heart of the communication between the back-end part of your game (that is the PHP side of your
code) and the front-end part (the Javascript part). Notifications are sent through websockets either to all users around
the table (players and spectators, so any public information: a players places a meeple on the board, or reveals a card)
or to a specific player (since the back-end has no access to spectators around the table, you can only send information
to actual players, and this is for private information: the card a player has drawn for example).

#### Anatomy of a notification

There are four elements to a notification:
1. The first is the channel (or websocket) the notification is sent to. Basically, a notification sent to all players is
   sent once per user around the table, on each user's channel. Otherwise, a private notification is sent only on the relevant
   player channel.
1. Then comes the notification name or identifier. This is a string that the developers choses to represent the
   notification, and to attach the relevant handler on the JS side.
1. Here is one of the most important part of the notification: the notification log message. This is the message that
   will appear in the game log and that will go through the translation process (which I'll cover further down). This message
   is a string defined with simple quotes `'` because it can contain substitution keys (for example `${cardName}` or
   `${player_name}`) which would be interpreted by PHP as variable names if the string was defined using double quotes `"`.
   The keys will be substituted on the JS side using the `format_string_recursive` (which we'll also cover further down)
   using the values in the array that is the next and last part of the notification:
1. The last part is a PHP associative array, usually called the notification arguments (nicknamed args). All the keys of
   the log message string must be present as keys in the args associative array. You can obviously include more keys in that
   array if you need to provide information to the front-end which doesn't need to appear in the log message (for example
   values to update the player panel, player score, and so on).

These four parts are what make up a notification. One information of note: the notification log message can be empty.
This allows you to transmit information to the front-end without appearing in the log. This can be useful in a multistep
action to allow for UI update for the player carrying out the action without showing anything in the game log for example.

You may have read about recursive notifications in the BGA studio wiki, but I'm keeping this under wraps until we reach
the part describing the `format_string_recursive` method. Frustrating, isn't it? I know. But I try to step up the complexity
continuously during this article, and we're still at the basics so...

#### When and how are notifications sent ?

Notifications are sent only if the request made by the front-end is carried out without exceptions. That is to say, only
if the action (in the framework sense) was carried out successfully. Once the request is entirely processed, all the
notifications are sent in one package, in the order they were created. Also they will be processed on the JS side in the
same order they were sent. One thing you may need to be aware of: there seems to be a limit to the size of the notification
package sent to the front-end: the package cannot exceed 128kb. If you go over this limit, you'll have to take extra
steps that are outside the scope of this article.

The notifications are always sent from your PHP code, and received by the JS code, never the other way around. You can
check the [wiki](https://en.doc.boardgamearena.com/Main_game_logic:_yourgamename.game.php#Notify_players) to find out
about these methods. Be aware that these methods are not static despite the examples shown in the wiki, so you should
actually use the following syntax:

```php
$this->notifyPlayer($player_id, $notification_type, $notification_log, $notification_args);
\\And
$this->notifyAllPlayers($notification_type, $notification_log, $notification_args);
```

#### Notifications and translation

Since most of the time, your notifications will have a log message associated to them, you need to make sure that this
log message is translated. To do this, you simply need to pass it as an argument to the `clienttranslate` function. So
your `$this->notifyPlayer` and `this->notifyAllPlayers` calls will most likely look like this:

```php
$this->notifyAllPlayers('gainCoins', clienttranslate('${player_name} gains ${coin_number} coins'), [
    "player_name" => $this->getActivePlayerName(),
    "coin_number" => 5,
]);
$this->notifyPlayer($playerId, 'drawCardSelf', clienttranslate('You drew a card'), []);
```

Yes as you can see the notification arguments can be empty. Because there are no parameters in my notification log message.

A bit of vocabulary as well here. In the `$this->notifyAllPlayers` call, the string is a bit special as it contains
*parameters* (the parts like `${player_name}` and `${coin_number}`), or as we have already called them *substitution keys*. They
are actually referred to as keys since they have to be keys of the args associative array. And the value of the key is
the value linked to that specific key in the args array (so the `$args["key"]` value). That point is rather important for
another part of this article, so you should make sure you have the right definitions for those words.

Also, I've been a bit quick on how `clienttranslate` really works. Its role is not to actually translate the string. It's
there to tell the translation system that that specific string will need to be translated. But it will just return the
original string without translating it. If you think this is confusing, then buckle up, it will probably get weirder in
a few lines.

But the former paragraph will serve as a good transition, as now we'll leave (for the moment) the nice world of
notifications to enter the murkier waters of translation.

### Translation

Translation can be the bane of BGA developers. In my humble opinion, this happens because of the lack of explanation in
the wiki about the internals of translation.

#### Where does translation happen ?

Most (like 80 to 90%) of the translation will happen on the client side, that is to say, in JS. The only strings that are
actually going to be translated by the BGA servers are the strings you use in the `X.view.php` file, the strings that
are passed as arguments to the `totranslate` function in `gameoptions.inc.php`, `gameinfos.inc.php` and `stats.inc.php`, and
the strings passed through the `self::_()` method when writing exception messages in `X.game.php`. Otherwise, everything
will happen on the JS side, through the use of the `_()` function. We'll come back to that specific function later.

#### What all those functions do ?

What each of these functions (be them PHP or JS functions) do is mark the string they receive as argument as translatable
and store those verbatim strings in a file that will be later interpreted by the translation system. And I do mean
verbatim. Basically, if the code consisted only of the notifications defined in the previous section, the translation
system would record only two strings:

* `'${player_name} gains ${coin_number} coins'`
* `'You drew a card'`

The example is voluntarily minimalistic so that everyone can understand what happens (in a real game, you'd have you game
state descriptions, a bunch of notification log messages, some UI strings, etc.). So those are the only two strings that
the translation system will ever know about. Period.

The function that actually have a *real* translation role are:

* On the server side (PHP):
  * `self::_()`
  * `totranslate`
* On the client side (JS):
  * `_()`

These will also fetch the corresponding translation in the translation files, and offer them to the client.

A quick word on how translation actually works. These functions will search for the exact string passed as argument in
the translation files (so those strings are actual translation keys). Should you modify any of them before the translation
system kicks in, then it will fail without a doubt. We'll come back to that specific point later. Why? Because these
functions will look for the string they're given as argument in the translation files, and return their translation in
the desired language.

So let's assume that we substituted the parameters `${player_name}` and `${coin_number}` with their value before the
translation system actually translates the string. The relevant translating function would actually receive a string that
would look like `'SwHawk gains 4 coins'`. It would then look through the translation file for that exact string (`'SwHawk
gains 4 coins'`) but wouldn't find it, as there are only the two strings stated in the previous section. So the translation
would fail, and any player would be presented with the english version of the string.

Instead, the substitution needs to happen after the translation. This way, translators can ensure that words actually come
in the correct order in the sentence. Yes. Think about languages that are written right to left instead of left to right,
or that have some strict word placement rules, like German, where the verb should always be positioned as the second
grammatical element of the sentence.

Performing the substitution after the translation ensures that the translated sentence will be correct in the player's
language. And you need to stick to that principle like glue. Otherwise, players will bug you about strings not being
translated, even though they may seem to appear translated in the translation system.

#### What if my parameters need to be translated ?

Yes I knew you'd ask the question. Of course, it stands to reason that your parameters may need translation, especially
in notification log messages. In order to achieve that, you need to tell the system that those string will need translation.
This can be done in three different ways, depending on where you're defining your string:

* On the PHP side, in notifications:
  ```php
  $this->notifyAllPlayers('notificationType', clienttranslate('${player_name} plays ${cardName}'), [
      "player_name" => $this->getActivePlayerName(),
      "cardName" => clienttranslate("My card Name"),
      "i18n" => [ "cardName" ],
  ]);
  ```
  By specifying the `"i18n"` key, I'm telling the system that the `cardName` value will need to be translated. Of course,
  in order to mark the string as translatable, I need to pass it through the `clienttranslate` function.
* On the PHP side, in exceptions:
  ```php
  throw new BgaUserException(sprintf(self::_("You selected %s which is a %s card, and you can only select %s cards"),
      self::_("A card name"),
      $cardType, // defined using self::_("")
      $wantedCardType)); // also defined using self::_("")
  ```
* On the JS side:
  ```js
  dojo.string.substitute(_("${cardName} costs ${cost} coins and you only have ${amount} coins"), {
    cardName: _("Some card name"), // or any variable defined using _()
    cost: cost,
    amount: amount,
  })
  ```

#### What if I need to procedurally generate a sentence in my notification log messages ?

Yes the above examples may seem complex, but most probably, you'll find yourself in a situation were you need to generate
a translatable string procedurally. You will need the functionality offered by the `format_string_recursive` method we'll
cover in just a bit. But Timothée Pecatte has written a
[wonderful in-depth post about procedural notification]({% post_url 2021-11-28-translations-summary %}), which I
strongly encourage you to read.

Again, what a nice transition to the next section of this article:

### `format_string_recursive` method

The post linked in the previous section goes in-depth about of the function works, even though the code has been simplified
a bit in the process. This method always receives two arguments: the string (referred to as *log*, since most of the time it
will process your notification **log** message) where the substitution needs to happen, and the object which holds the
values that are going to be substituted (referred to as *args*, as most of the time, it will be your notification **args**).
Very roughly put, the method does the following:

1. It translates the log string (see translation process above)
1. It checks for the presence of the "i18n" property in the args object.
  1. If present, then all the values associated to the keys in the "i18n" array are substituted by their translation.
1. It then goes through all the keys, skipping the "i18n" array:
  1. If the value associated to that key is an object having properties named `log` and `args` and that neither of
     those properties are undefined and that the `args` property is indeed an object, that specific value will be
     substituted by the result of applying the `format_string_recursive` function to the `log` property using the `args`
     property as args.
1. Then it substitutes each key in the string with the corresponding value in the args object using the
   `dojo.string.substitute` method from the dojo toolkit. This specific method has some nice functionnality, which we'll
   cover in a few shakes.

Once again, if this is not enough to your liking, I strongly encourage you to read
[this post]({% post_url 2021-11-28-translations-summary %}) which will show you excerpts of the `format_string_recursive`
method's source code.

Let's assume our PHP backend sends the following notification:

```php
$this->notifyAllPlayers('discardCards',clienttranslate("${player_name} discards ${cardNames}"), [
    "player_name" => $this->getActivePlayerName(),
    "cardNames" => [
        "log" => "${cardName0}, ${cardName1}, ${cardName2}",
        "args" => [
            "i18n" => [ "cardName0", "cardName1", "cardName2" ],
            "cardName0" => clienttranslate("A card name"),
            "cardName1" => clienttranslate("Another card name"),
            "cardName2" => clienttranslate("A last card name"),
        ],
    ],
]);
```

Here's how `format_string_recursive` would treat it, should the player be french:

0. The notification object looks like this:
   ```json
   {
       "type": "discardCards",
       "log": "${player_name} discards ${cardNames}",
       "args": {
           "player_name": "SwHawk0",
           "cardNames": {
               "i18n": [ "cardName0", "cardName1", "cardName2" ],
               "cardName0": "A card name",
               "cardName1": "Another card name",
               "cardName2": "A last card name"
           }
       }
   }
   ```
1. Translating the log string: `"${player_name} se défausse de ${cardNames}"`
1. There is no "i18" property in the args object, so no further translation need to happen at this step
1. Looping through all the keys:
   1. player_name isn't an object, going to the next key
   1. cardNames is an object (remember PHP associative arrays become JS objects), and it has a log property, and an args
      property that is an object. So `format_string_recursive` is applied to this key:
      1. Translating the log string: `"${cardName0}, ${cardName1}, ${cardName2}"`, actually, no translation happens here,
         since the string hasn't been marked as trasnlatable (clienttranslate wasn't called).
      1. There's a "i18n" property:
         1. The value associated with cardName0 is changed to: "Un nom de carte"
         2. The value associated with cardName1 is changed to: "Un autre nom de carte"
         3. The value associated with cardName2 is changed to: "Un dernier nom de carte"
      1. All the keys are not objects, so no further call to `format_string_recursive`
      1. All the keys are substituted by their value: `"Un nom de carte, Un autre nom de carte, Un dernier nom de carte"`
   1. cardNames value is changed to : `"Un nom de carte, Un autre nom de carte, Un dernier nom de carte"`, at that point
      the notification object looks like:
      ```json
      {
          "type": "discardCards",
          "log": "${player_name} se défausse de ${cardNames}",
          "args": {
              "player_name": "SwHawk0",
              "cardNames": "Un nom de carte, un autre nom de carte, un dernier nom de carte"
          }
      }
      ```
1. All the keys are now substituted by their value: `"Swhawk de défausse de Un nom de carte, Un autre nom de carte, Un dernier nom de carte"`

The information of note here is that the substitution is actually performed by a dojo method, whose documentation can be
found [here](https://dojotoolkit.org/reference-guide/1.7/dojo/string.html) (I'll let you be the judge of the quality of
this documentation), and whose source code can be examined
[here](https://github.com/dojo/dojo/blob/e3fa44ff3b509c779a420e6b39af54522fa9890c/string.js#L61) (I do find the comments
in the source code to be more enlightening about how this function works than the documentation). This method can
substitute keys by their value (provided they are presented like this `${key}`, but the third and fourth arguments allow
to format the value in any way we want.

Which will be quite useful for the next part of this article:

### Javascript HTML log injection
