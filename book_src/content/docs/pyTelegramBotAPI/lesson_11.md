---
title: "Урок 11"
description: "Ведём (более-менее) осмысленные диалоги. Конечные автоматы"
type: docs
url: "/docs/lesson_11"
BookToC: true
weight: 12
---

## Общая часть
Полезных ботов мы уже умеем делать (если вам вдруг так не кажется, перечитайте первые уроки), теперь пора делать, кхм, «смышлённых».

Но для начала – немного теории. Будем считать, что диалог пользователя с ботом можно разбить на логические части: начало диалога, запрос одной информации, запрос другой информации, возврат к определенному этапу диалога, конец диалога. При этом можно сказать, что при каждом сообщении пользователя, его состояние (или состояние бота, с какой стороны посмотреть) меняется, и между этими состояниями осуществляются переходы. Под «состоянием» будем понимать следующее ожидаемое от пользователя действие.  
Наш бот должен иметь возможность определять текущее состояние пользователя, подбирать соответствующие сообщения, а также дожидаться нужного ответа. Поставим задачу следующим образом: бот запрашивает у пользователя имя, возраст и просит отправить картинку. Не допускается некорректный ввод возраста и пропуск какого-либо шага. По команде /reset диалог начинается заново. Состояние пользователя не должно потеряться при перезагрузке бота.  
На следующем рисунке изображены возможные состояния бота и переходы между ними.

{{% img2 src="/images/l11_1.png" caption="Переход между состояниями" %}}

Бот должен помнить все сохранённые состояния даже после перезагрузки, поэтому нам потребуется отдельное хранилище во внешней памяти (например, на жёстком диске). Будем использовать однофайловую БД [Vedis](https://vedis-python.readthedocs.io/en/latest/), позволяющую удобно хранить пары «ключ-значение». В качестве ключа возьмём ID пользователя, конвертированный в строку, а в качестве значения - его «состояние».

## Пишем бота
С базы и начнём. Создадим пустой файл `bot.py` и рядом с ним `dbworker.py`, в котором опишем два метода для работы с БД: получение текущего состояния и смена состояния на желаемое.

```python
# -*- coding: utf-8 -*-

from vedis import Vedis
import config

# Пытаемся узнать из базы «состояние» пользователя
def get_current_state(user_id):
    with Vedis(config.db_file) as db:
        try:
            return db[user_id].decode() # Если используете Vedis версии ниже, чем 0.7.1, то .decode() НЕ НУЖЕН
        except KeyError:  # Если такого ключа почему-то не оказалось
            return config.States.S_START.value  # значение по умолчанию - начало диалога

# Сохраняем текущее «состояние» пользователя в нашу базу
def set_state(user_id, value):
    with Vedis(config.db_file) as db:
        try:
            db[user_id] = value
            return True
        except:
            # тут желательно как-то обработать ситуацию
            return False
```

Как видно из кода выше, не хватает ещё файла `config.py`. Создадим этот файл, в нём укажем токен бота, название базы данных (с расширением `.vdb`) и зададим класс со списком возможных состояний пользователя:

```python
# -*- coding: utf-8 -*-

from enum import Enum

token = "1234567:ABCxyz"
db_file = "database.vdb"


class States(Enum):
    """
    Мы используем БД Vedis, в которой хранимые значения всегда строки,
    поэтому и тут будем использовать тоже строки (str)
    """
    S_START = "0"  # Начало нового диалога
    S_ENTER_NAME = "1"
    S_ENTER_AGE = "2"
    S_SEND_PIC = "3"
```

Настало время перейти к описанию логики бота. По команде **/start** будем инициировать начало диалога и спрашивать у юзера его имя, затем переключать «состояние» на «ожидаем ввода имени». По команде **/reset** будем возвращаться в начало диалога, спрашивать имя и т.д., копируя код из обработчика **/start**. Различия появятся позже.

```python
# -*- coding: utf-8 -*-

import telebot
import config
import dbworker

bot = telebot.TeleBot(config.token)

# Начало диалога
@bot.message_handler(commands=["start"])
def cmd_start(message):
    bot.send_message(message.chat.id, "Привет! Как я могу к тебе обращаться?")
    dbworker.set_state(message.chat.id, config.States.S_ENTER_NAME.value)

# По команде /reset будем сбрасывать состояния, возвращаясь к началу диалога
@bot.message_handler(commands=["reset"])
def cmd_reset(message):
    bot.send_message(message.chat.id, "Что ж, начнём по-новой. Как тебя зовут?")
    dbworker.set_state(message.chat.id, config.States.S_ENTER_NAME.value)
```

Теперь нам нужен хэндлер, который сработает только при определённом состоянии пользователя. Отлично, прямо так и сделаем:

```python
@bot.message_handler(func=lambda message: dbworker.get_current_state(message.chat.id) == config.States.S_ENTER_NAME.value)
def user_entering_name(message):
    # В случае с именем не будем ничего проверять, пусть хоть "25671", хоть Евкакий
    bot.send_message(message.chat.id, "Отличное имя, запомню! Теперь укажи, пожалуйста, свой возраст.")
    dbworker.set_state(message.chat.id, config.States.S_ENTER_AGE.value)
```

Обратите внимание: мы сравниваем текущее состояние пользователя со значением состояния, необходимым для входа в функцию. Если у пользователя в данный момент другое состояние, то подхэндлерный метод просто не вызовется. Соответственно, если у вас два хэндлера, реагирующих на одно и то же состояние, сработает первый по списку.

Следующая функция должна принять от пользователя его возраст. Если в первом случае нам было всё равно, то теперь придётся заняться проверкой вводимых значений, причём делать надо именно на втором шаге, а не на первом.

```python
@bot.message_handler(func=lambda message: dbworker.get_current_state(message.chat.id) == config.States.S_ENTER_AGE.value)
def user_entering_age(message):
    # А вот тут сделаем проверку
    if not message.text.isdigit():
        # Состояние не меняем, поэтому только выводим сообщение об ошибке и ждём дальше
        bot.send_message(message.chat.id, "Что-то не так, попробуй ещё раз!")
        return
    # На данном этапе мы уверены, что message.text можно преобразовать в число, поэтому ничем не рискуем
    if int(message.text) < 5 or int(message.text) > 100:
        bot.send_message(message.chat.id, "Какой-то странный возраст. Не верю! Отвечай честно.")
        return
    else:
        # Возраст введён корректно, можно идти дальше
        bot.send_message(message.chat.id, "Когда-то и мне было столько лет...эх... Впрочем, не будем отвлекаться. "
                                          "Отправь мне какую-нибудь фотографию.")
        dbworker.set_state(message.chat.id, config.States.S_SEND_PIC.value)
```

Как видно из скриншота ниже, при вводе некорректных значений бот не сбрасывает диалог и не переходит к следующим вопросам, а «удерживает» состояние, вынуждая пользователя ответить корректно, при этом на шаге №1 любой ввод позволял перейти далее.

{{% img2 src="/images/l11_2.png" caption="Некорректный ввод" %}}

Наконец, на последнем шаге мы ожидаем отправку изображения, поэтому дополнительно выставляем нужный `content_types`:

```python
@bot.message_handler(content_types=["photo"],
                     func=lambda message: dbworker.get_current_state(message.chat.id) == config.States.S_SEND_PIC.value)
def user_sending_photo(message):
    # То, что это фотография, мы уже проверили в хэндлере, никаких дополнительных действий не нужно.
    bot.send_message(message.chat.id, "Отлично! Больше от тебя ничего не требуется. Если захочешь пообщаться снова - "
                     "отправь команду /start.")
    dbworker.set_state(message.chat.id, config.States.S_START.value)
```

Если теперь запустить бота и проверить, логика должна быть правильной: на каждом этапе бот ожидает от юзера конкретное действия, возможно, проверяя корректность ввода. По команде **/reset** сбрасывает в начало, а благодаря записи текущего состояния на диск, боту не страшны перезапуски. Остаётся одна маленькая деталь: вдруг пользователь случайно очистит диалог с ботом или вдруг приложение заглючит и придётся снова вызывать команду **/start**. Добавим в первый обработчик несколько проверок, чтобы после долгой разлуки бот продолжал общаться с юзером на том месте, где они остановились:

```python
# Начало диалога
@bot.message_handler(commands=["start"])
def cmd_start(message):
    state = dbworker.get_current_state(message.chat.id)
    if state == config.States.S_ENTER_NAME.value:
        bot.send_message(message.chat.id, "Кажется, кто-то обещал отправить своё имя, но так и не сделал этого :( Жду...")
    elif state == config.States.S_ENTER_AGE.value:
        bot.send_message(message.chat.id, "Кажется, кто-то обещал отправить свой возраст, но так и не сделал этого :( Жду...")
    elif state == config.States.S_SEND_PIC.value:
        bot.send_message(message.chat.id, "Кажется, кто-то обещал отправить картинку, но так и не сделал этого :( Жду...")
    else:  # Под "остальным" понимаем состояние "0" - начало диалога
        bot.send_message(message.chat.id, "Привет! Как я могу к тебе обращаться?")
        dbworker.set_state(message.chat.id, config.States.S_ENTER_NAME.value)
```

Теперь вы умеете контролировать диалог пользователя и бота. Не забудьте изучить [полный код урока](https://github.com/MasterGroosha/telegram-tutorial/tree/master/lesson_11) на Github, если что-то осталось непонятно.

P.S. Пара ссылок про конечные автоматы:  
1. https://tproger.ru/translations/finite-state-machines-theory-and-implementation/  
2. https://habrahabr.ru/post/160105/

{{< btn_left relref="/docs/pytelegrambotapi/lesson_10" >}}Урок №10{{< /btn_left >}}
{{< btn_right relref="/docs/pytelegrambotapi/lesson_12" >}}Урок №12{{< /btn_right >}}