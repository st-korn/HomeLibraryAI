# Quick and simple catalog of the home library, <br/>based on telergam-bot with AI-recognition <br/>of covers and annotations

[![HomeLibraryAI Avatar](images/avatar_min.jpg)](images/avatar.jpg) ![HomeLibraryAI bot QR-code](images/t_me-home_library_ai_bot.png)


Telegram bot to photo the covers and annotations of your books and create a catalog of your home library. Share the catalog with your friends. No manual input: AI will do everything for you, and it will take no more than 10 seconds to enter one book. If necessary, you can always download your collection to a file. [@home_library_ai_bot](https://t.me/home_library_ai_bot)

## Basic usage

Just start chatting with [@home_library_ai_bot](https://t.me/home_library_ai_bot) in telegram. We use `Telegram ID` to identificate user and store it's books. Your telegram ID is permanent and does not change when you changing mobile number or telegram nickname.

Cathegories are not stored in database as a separate entity. They collect everytime from stroed books of the user. Then user add new cathegory, it stored in users variable and applied to the new book. Then just one book with this cathegory saved, we will start receiving this cathegory from selection cathegories of all users books.

We store all your photos with unique anonymous identifiers is S3 file storage. So if someone known the photo identificator - they can see it. Access to other people's photos is unlikely, but try not to photograph things that you would not like to allow for public review.

For each book we need two photos:
- Photo of book's cover - for extract cover's picture and store them in library databas
- Photo of first book's page with annotation - for extract from them text imformation about the book

## AI processing of photo of book's cover 

Photo of the book cover on plain surface, for example, on the desk. Use the desktop lamp or mobilephone flash to illuminate the book. Avoid mirrored or glass surfaces. Try not to use tables with a colorful surface, such as wooden ones. Hold the phone at a right angle to the table to avoid trapezoidal deformations.

We use [U-2-Net Salient Object Detection AI-model](https://github.com/xuebinqin/U-2-Net) of the [python rembg library](https://github.com/danielgatis/rembg) to remove background. Next we use [OpenCV library](https://opencv.org/) to find the cover's rectanlge and them align it in the form of a vertical rectangle.

| Source photo | Extracted cover |
| - | - |
| [![Example 1 - source](images/th_cover1.jpg)](examples/find_cover/cover1.jpg) | [![Example 1 - result](images/th_output1.jpg)](examples/find_cover/output1.jpg) |

Step by step example:

| Source photo | → | Photo without backgroud | → | Convert to grayscale and add some blur | → |
| - | - | - | - | - | - |
| [![Source photo](images/th_cover4.jpg)](examples/find_cover/cover4.jpg) |  → | [![Photo without backgroud](images/th_debug00.jpg)](examples/find_cover/step-by-step/debug_00_without_cover.png) | → | [![Convert to grayscale and add some blur](images/th_debug03.jpg)](examples/find_cover/step-by-step/debug_03_blurred.jpg) | → |

| → | Highlight all edges | → | Dilate edges | → | Erode edges | → |
| - | - | - | - | - | - | - |
| → | [![Highlight all edges](images/th_debug04.jpg)](examples/find_cover/step-by-step/debug_04_edges.jpg) |  → | [![Dilate edges](images/th_debug05.jpg)](examples/find_cover/step-by-step/debug_05_dilated.jpg) | → | [![Erode edges](images/th_debug06.jpg)](examples/find_cover/step-by-step/debug_06_eroded.jpg) | → |

| → | Find all contours | → | Select the largest rectangled contour | → | Do perspective transformation |
| - | - | - | - | - | - |
| → | [![Find all contours](images/th_debug07.jpg)](examples/find_cover/step-by-step/debug_07_all_contours.jpg) |  → | [![Select the largest rectangled contour](images/th_debug09.jpg)](examples/find_cover/step-by-step/debug_09_rectangle_1.jpg) | → | [![Result](images/th_output4.jpg)](examples/find_cover/output4.jpg) |

## AI processing of photo of book's annotation

Take a picture of the first annotation page of the book in the same way. Inevitably, your fingers holding the book will get into the photo. If they don't obscure the text, it's okay.

We use an external LLM model for OCR the first annotation page of the book to text and extract from them some important fields about the book. 

We exctract these fields:
| Field | Value |
| - | - |
| `title` | Book title |
| `authors` | Authors of the book |
| `authors_full_names` | Full names of the book's authors|
| `pages` | Pages count in the book |
| `publisher` | Organiozation name of book's publisher |
| `year` | Year of publish the book |
| `ISBN` | International Standard Book Number, if exists |
| `annotation` | Full text of annotation, extracted from the page |
| `brief` | Brief summary of the annotation |

The following prompt for processing a photo by AI-model I found the best:

<blockquote><code>The photo contains a page with the book's annotation.
Your response should only consist of the [book] section of an ini file without any introductory or concluding phrases.
The section must always contain 9 parameters. If a parameter is missing, its value should be empty.
Provide the parameter values in the same language as the photographed page. 
If the parameter value contains newlines or equal signs, replace them with spaces.
Output only plain text of ini file content. Do not output markdown.
Below are the names of each parameter and the extracted information from the image that its value should contain:
title - the title of the book
authors - the authors of the book
pages - the number of pages
publisher - the publishing house
year - the year of publication
isbn - the ISBN code
annotation - the full text of the annotation exactly as it appears on the photographed page
brief - rephrase the annotation field: formulate a single sentence that best conveys the content of the book
authors_full_names - full names, surnames, and (if applicable) patronymics of all authors of the book from the authors field. The page likely contains mentions of their full names. Find and provide them in this field
</code></blockquote>

Despite all efforts, models regularly:
- leave markdown formatting in the response
- forget to remove newlines from long field values
- forget to include the title section [book] in the response. 

All these options should be considered during the response processing process.

During the experiments, it turned out that the result of the model depends on the order of enumeration of the extraced parameters. Such derived parameters as `brief` and `authors_full_names` should go after all other parameters:

- If we try to extract `brief` before full `anotation`, we couldn't get the model to get really full text of book's annotation in `annotation` field.
- When model construct `authors_full_names`, it can add their own knowledge and append middle name or decipher the initials - even there are no such information on the source page.

Interesting observation:
- in 4 out of 5 cases AI-model return `Dale Carnegie` authors name, but in 1 case it return `Carnegie Dale`.

We test 28 LLM's which were relevant in May 2025. To each model we three pages:

| Source photo | Extracted fields |
| - | - |
| [![Example 1 - source](images/th_page1.jpg)](examples/extract_book_info/page1.jpg) | `name` = Птицы на кормушках: Подкормка и привлечение <br/> `authors` = Василий Вишневский<br/> `authors_full_names` = Вишневский Василий Алексеевич <br/> `pages` = 304 <br/> `publisher` = Фитон XXI <br/> `year` = 2025 <br/> `ISBN` = 978-5-6051287-5-5 <br/> `annotation` = Подкормка птиц — очень важное и нужное дело, а наблюдение за пернатыми посетителями кормушек приносит массу удовольствия. Книга даёт исчерпывающие ответы на самые важные вопросы: как, чем, когда и каких диких птиц подкармливать. Делать это можно в самых разных местах: от балкона городской квартиры до дачного участка, парка или близлежащего леса. Большое внимание в книге уделено разнообразию кормов, и поскольку не все они полезны, то и тому, чем можно (а чем нельзя) кормить, как приготовить корм, как его хранить. Кроме подкормки, вы можете посадить определённые растения, которые привлекут ещё больше птиц на дачный участок. Поскольку посещение кормушек связано с рядом опасностей, в книге даны очень важные рекомендации, как их избежать и как защитить пернатых гостей от врагов и конкурентов. Если вы повесили кормушку, но никто на неё не прилетает - загляните в главу о том, как привлечь птиц в этом случае. И наконец, большая заключительная часть книги посвящена разнообразию птиц, которых можно подкармливать: как самых обычных, так и редких всем необходима ваша помощь в трудные времена. <br/> `brief` = Книга о том, как правильно подкармливать птиц зимой. `⬅️ Note: this thesis is independently formulated by the GPT-model ‼️` |
| [![Example 2 - source](images/th_page2.jpg)](examples/extract_book_info/page2.jpg) | `name` = Асимптотические методы для линейных обыкновенных дифференциальных уравнений <br/> `authors` = Федорюк М. В.<br/> `authors_full_names` = Михаил Васильевич Федорюк <br/> `pages` = 352 <br/> `publisher` = Наука <br/> `year` = 1983 <br/> `ISBN` = <br/> `annotation` = В книге содержатся асимптотические методы решения линейных обыкновенных дифференциальных уравнений. Рассмотрен ряд важных физических приложений к задачам квантовой механики, распространения волн и др. Для математиков, физиков, инженеров, а также для студентов и аспирантов университетов и инженерно-физических вузов. <br/> `brief` = В книге содержатся асимптотические методы решения линейных обыкновенных дифференциальных уравнений. |
| [![Example 3 - source](images/th_page3.jpg)](examples/extract_book_info/page3.jpg) | `name` = Как завоёвывать друзей и оказывать влияние на людей <br/> `authors` = Д. Карнеги<br/> `authors_full_names` = Дейл Брекенридж Карнеги `⬅️ Note: authors name has been updated with information from Wikipedia by Gemini Pro Vision Preview model ‼️` <br/> `pages` = 352 <br/> `publisher` = Попурри <br/> `year` = 2013 <br/> `ISBN` = 978-985-15-1966-4 <br/> `annotation` = Поучения, инструкции и советы Дейла Карнеги за десятки лет, прошедшие с момента первого опубликования этой книги, помогли тысячам людей стать известными в обществе и удачливыми во всех начинаниях. Наследники автора пересмотрели и немного обновили текст, подтверждая его актуальность и теперь, в начале нового века. <br/> `brief` = Книга Дейла Карнеги о том, как заводить друзей, оказывать влияние на людей и добиваться успеха в жизни. |

I try to ask each of 28 GPT vision models, aviable on May 2025 to process each of these 3 pages.

For this operation I decided to write python script without any coding - only using GPT-models. I use `Clause 3.7 Sonnet` model and start from simple prompts. I try to generate code, found and analyze errors, add some lines to prompt, try again, etc..

Resulting prompt, witch generate full-functionaly successefull worked script is:

<blockquote><code>1. Read GPT_URL, GPT_API_TOKEN, GPT_MODEL from environment variables.
2. Use AsyncOpenAI for GPT client
3. Read models list from models.txt. Sort them asc
4. Read prompt from prompt.txt
5. Send all *.jpg pictures to each model with readed prompt
6. Response from each model must be a text of an ini-file with one section [book] and values of 9 parameters: title, authors, pages, publisher, year, isbn, annotation, brief, authors_full_names.
7. Create excel file with sheet for each of 9 parameters. In rows of the sheet must be models in columns - pictures, in cells - readed from model values of these parameters.
8. If model does not get response, or response is not right ini-file or you can not find parameter in response - put "-" char to the appropriate cell.
9. Remove markdown start and and of code blocks from model's result.
10. If you found line withoput equals character - ignore them.
11. Pay attention: for .txt, .jpg and result excel file you must use not current system folder, but folder of your python program
12. Write response of each model on each picture to .txt file in 'debug' subfolder with name of file contains model and picture names. Note that model names can include characters, witch can not be in file names. Create the 'debug' subfolder if they does not exists.
13. Add to program Boolean ONLYONE parameter, then it runs full, but process only first model and only first picture.
14. Output on the screen information about current model and file then process.
15. Store time (count of seconds) of answer for each model and each picture and store them in such sheet 'time' in result excel table.
</code></blockquote>

Generated script you can see in [compare_models.py](examples/extract_book_info/compare_models.py)

Then I calculate count of right answers for each of 3 pages and each of 9 parameters. If the answer is right and full model get `+1 score`. Maximum models can have `27 scores`. It means, that this model can be used for page recognition. Only 11 of 28 models have never made a mistake. Also I count elapsed time (in seconds) for each request and summarize them for models. And I count manual cost of use models ($USD of 1000 request). 

Final results of this comparsion are here:

![GPT models comparison](images/model_comparison.png)

Summary:

1. We found two best models to use: `Gemini Flash 1.5` and `Gemini Flash 2.0` by Google. They are the cheapest and allways right. 
2. `Gemini Flash 2.5` is yet on preview stage and have errors. Normal version of this model, once returned answer not on book language. And thinking version get an hallutination on ISBN field - but where are no such field in book, published in 1983.
3. `Gemini Pro 1.5` is the good model, but a little expensive. Early I use its cheaper instance `Gemini Pro Vision Preview` - but Google closed it at May 2025
4. `Claude 3 Opus`, `Claude 3.5 Sonnet` and `Clause 3.7 Sonnet` by Anthropic are good, but fantastic expensive models.
5. OpenSource `Gemma 3` by Google showed the worst result, as well as `Llama 3` by Meta.
6. Surprisingly good result get `Llama 4 Maverick`. It is a little slow, but you can run it on your own server, and do not pay for using GTP's for more! A good option, if you don't have a money but have one free NVidia H100 GPU-card.
7. `GPT-4 Turbo` by OpenAI failed this test. But all other OpenAI's models works fine.
8. I recommend to use only `GPT-4o Mini` model by OpenAI. It has an equivalent in price to `Gemini Pro 1.5` model. The models `GPT-4o` and `o4` by OpenAI are too expensive.

Resume: In my production I will use `Gemini Flash 1.5` model.

## Telegram-bot's dialog algorithm

![Telegram-bot's dialog algorithm](images/algorithm.drawio.png)

Here is [the first handwritten edition](images/manuscript.pdf) of this algorithm.

## States of the bot

- `wait_for_command` - waiting for the one of global bot's commands
- `select_lang` - waiting for user to select one of languages
- `select_cathegory` - waiting for user to select a cathegory or enter the new one
- `wait_for_cover_photo` - waiting for user to send a photo of the book cover
- `wait_reaction_on_cover` - waiting for user's reaction of extracted book cover
- `wait_for_brief_photo` - waiting for user to send a photo of the annotation page
- `wait_reaction_on_brief` =  waiting for user's reaction of extracted book information

## Telegram bot's commands and handlers

There are 5 global bot's commands, whitch can be executate from any bot state. You don't need to ask `@BotFather` add these commands to main menu button. They added automaticaly by the bot itself:
- `add` - Add book
- `find` - Search
- `edit` - Edit book
- `cat` - Cathegories
- `export` - Export
- `lang` - Language

Also bot process `/start` command - for the first run of each user.

## User's data accumulating and stored for each bot's session

Common data:

- `locale`: str - prefered language by user's selection
- `inline`: int - last ID of message with inline keyboard. Used to remember remove these keyboard, when they are no longer needed

Cathegory selection:

- `action`: str - name of the action to run after the user selects a category
- `can_add`: bool - can the user enter the name of a new category when he select it
- `cathegory`: str - name of the cathegory, selected by user

Book data:
- `photo_filename`: str - relative path to file with original photo on S3 storage of the book cover
- `cover_filename`: str - relative path to file with extracted photo on S3 storage of the book cover
- `brief_filename`: str - relative path to file with original photo on S3 storage of the annotation page
- `title`: str - book's title
- `authors`: str - authors of the book
- `authors_full_names`: str - full names of authors of the book
- `pages`: str - count of pages in the book
- `publisher`: str - the publishing house
- `year`: str - year of the book publication
- `isbn`: str - ISBN
- `brief`: str - single sentence that best conveys the content of the book
- `annotation`: str - full text of the book's annotation
- `book_id`: int - ID of added/edited book
- `user_id`: int - ID of current telegram-user (library owner)

## How it works

![Working environment diagram](images/homelib.drawio.png)

## GitHub actions secrets

| Name | Description | Usage | Example |
| - | - | - | - |
| REGISTRY_HOST | Hostname of Container Registry to push docker image | CI/CD | `192.168.110.157` |
| REGISTRY_USERNAME | Login of Container Registry | CI/CD | `76ee9321-2...` |
| REGISTRY_PASSWORD | Password of Container Registry | CI/CD | `1287a999-f...` |
| KUBECONFIG | YAML text config of production Kubernates cluster to deploy docker container | CI/CD | `apiVersion: v1`<br/>`clusters:`<br/>`- cluster:`<br/>... |
| TELEGRAM_TOKEN | Strint token for production telegram-bot @home_library_ai_bot | Production | `25461226:Fjkld876ww2...` |
| POSTGRES_HOST | Hostname or IP-address of Postgres database server | Production | `127.0.0.1` |
| POSTGRES_PORT | IP-Port of Postgres database server | Production | `5432` |
| POSTGRES_DATABASE | Database name of Postgres database | Production | `homelib` |
| POSTGRES_USERNAME | Login of Postgres database | Production | `user` |
| POSTGRES_PASSWORD | Password of Postgres database | Production | `my_super_password` |
| AWS_ENDPOINT_URL | URL of S3 storage | Production | `https://s3.ru-7.storage.selcloud.ru` |
| AWS_BUCKET_NAME | Bucket name in S3 storage | Production | `homelibrary` |
| AWS_ACCESS_KEY_ID | Access key to S3 storage | Production | `e8793d292328x...` |
| AWS_SECRET_ACCESS_KEY | Secret key to S3 storage | Production | `a8632409c821...` |
| GPT_URL | URL for access to GPR API | Production | `https://api.vsegpt.ru/v1` |
| GPT_API_TOKEN | Secret token for GPT API | Production | `sk-f3-wm-15a2432133...` |
| GPT_MODEL | GPT model name | Production | `vis-google/gemini-flash-1.5` |

Add production secrets they are delivered to the application via environment variables. If you use `Visual Studio Code` to develop, you can add them to `.vscode/launch.json` file:
```
    "configurations": [
        {
            "env": {
                "TELEGRAM_TOKEN": "25461226:Fjkld876ww2...",
                "POSTGRES_HOST": "127.0.0.1",
                "POSTGRES_PORT": "5432",
                "POSTGRES_DATABASE": "homelib",
                "POSTGRES_USERNAME": "user",
                "POSTGRES_PASSWORD": "my_super_password",
                "AWS_ACCESS_KEY_ID": "e8793d292328x...",
                "AWS_SECRET_ACCESS_KEY": "a8632409c821...",
                "AWS_ENDPOINT_URL": "https://s3.ru-7.storage.selcloud.ru",
                "AWS_BUCKET_NAME": "homelibrary",
                "GPT_URL": "https://api.vsegpt.ru/v1",
                "GPT_API_TOKEN": "sk-f3-wm-15a2432133...",
                "GPT_MODEL": "vis-google/gemini-flash-1.5"
            }
        }
    ]
```

## PostgreSQL database

Where are two application tables in the PostgreSQL database:

1. Table `logs` for log of starting the bot:
```SQL
CREATE TABLE IF NOT EXISTS logs (
    user_id BIGINT,
    nickname TEXT,
    username TEXT,
    datetime TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

2. Table `books` for store information of all books, added by users:
```SQL
CREATE TABLE IF NOT EXISTS books (
    user_id BIGINT,
    book_id BIGINT,
    cathegory TEXT,
    photo_filename TEXT,
    cover_filename TEXT,
    brief_filename TEXT,
    title TEXT,
    authors TEXT,
    authors_full_names TEXT,
    pages TEXT,
    publisher TEXT,
    year TEXT,
    isbn TEXT,
    annotation TEXT,
    brief TEXT,
    datetime TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, book_id)
)
```

And two internal tables for continuous operation of the bot: one to store current state of conversation, and another to store current information, received from user. These tables used by `PostrgreSQL-storage` plugin, written by me for [`aiorgam`](https://aiogram.dev/) library.

3. Table `aiogram_states` for store current state of bot-user conversation
```SQL
CREATE TABLE IF NOT EXISTS aiogram_states(
    chat_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    state TEXT,
    PRIMARY KEY (chat_id, user_id)
)
```

4. Table `aiogram_data` for store information, received from user during the conversation:
```SQL
CREATE TABLE IF NOT EXISTS aiogram_data(
    chat_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    data JSON,
    PRIMARY KEY ("chat_id", "user_id")
)
```

You don't need to create these tables manualy. Then telegram-bot connect to postgres, it try to create these tables, if they are not exists.

## Project files

Program files:

- `homelib.py` - core of telegram-bot
- `modules\environment.py` - prepare environment variables, classes and connections
- `modules\databasecreation.py` - script to create tables in Postgres database on the first run of script
- `modules\handle_addbook.py` - handlers to for processing bot messages in adding book mode

Deployment scripts:

- `requirements.txt` - python's library dependencies
- `dockerfile` - instructions: how to build Docker container
- `deployment.yaml` - instructions: how to deploy it on Kubernates cluster
- `.gitignore` - hide my python cache, debug environment variables with sectets, certificates, etc.
- `.github\workflows\` - instructions: automatization CI/CD with GitHub Actions

Documentation:

- `README.md` - current description
- `images\` - floder with images for current description

## The knowledge I have earned

System administration and development organization:

- CI/CD with docker container generation and deploy it on kubernates cluster
- Using requirements.txt to build docker container with python's program
- Using GitHub secrets to deploy container and for transmission in it database coordinates and API keys
- Using Visual studio code launch.json to emulate environment variables with GitHub secrets
- Arrange branches: main to deploy on production and develop to store development changes

Cloud storages:

- Push files in S3 store, including asynchronous mode and get links to them

AI models:

- Use U-2-Net Salient Object Detection AI-model as local library. $0.1 - $1 per image economy!
- Automaticaly compare big count of different GPT models to find the cheapest and most efficient one
- Make requests to GPT AI-models, including images and parse their responses
- Write prompts for GPT to get scructurized data and get derived fields
- Write prompt on one language to get response on another language

AI code generation:
- Try to generate tiny but full-functionaly successefull worked program by one structurized request to GPT-model (see `generate_prompt_sonnet_3.7.txt` and `compare_models.py`)

Images processing:
- Find contoures on the image
- Affine transformation of the image

Intenational localization:

- Create multilanguage application and add or change localization without edit source code
- I realized that the best way is to store in program code only shortcuts for string literals. And all full strings, included default english language, should be stored in local files
- Plural numbers localizaion

Telegram development:

- Comparing different pythons framework for telegram bots. I chose aiogram for its beautiful structure and thoughtful concept
- Using aiogram FSMstore to remember state of conversation and user data, entered on previous steps of the dialogue
- Send formated messages in telegram
- Using inline buttons in telegram chat
- Timely removing of old, outdated inline buttons
- Put emoji reaction on messages and delete messages

Opensource collaboration:

- Wrote my own library to use PostgreSQL database to store state and user data on aiogram FSMstore

